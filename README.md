# `openvpn_pki` role

Creates and maintains the PKI for an OpenVPN server — a CA, the server's
certificate and key, a tls-crypt key, a certificate and key per client and a
certificate revocation list — plus a ready-to-import `.ovpn` profile for every
client. Everything is written as files **on the controller**, meant to be
committed to the inventory repository with the secrets encrypted by Ansible
Vault. The role keeps working on the encrypted files, so it can be re-run at
any time to add, revoke or renew clients.

What it deliberately leaves to other tools: **deploying the server** (the
[`homelab.openvpn`](https://github.com/shadowlordarholfromgothic2/ansible-role-openvpn)
role takes these files), **encrypting the secrets** (the role tells you
exactly what still needs `ansible-vault encrypt`) and handing the profiles to
their users.

## Requirements

* ansible-core ≥ 2.15
* `community.crypto` and `community.general` (also listed in
  [`requirements.yml`](requirements.yml)).
* The Python `cryptography` library on the controller. It comes with
  ansible-core.
* No inventory, no target host and no `become`: the role runs on the
  controller only (`delegate_to: localhost`, `run_once: true`, `become: false`).
  OpenVPN does not have to be installed there.

## What the role does

1. **Validates input** before writing anything: client entries (a `name`
   that is safe as a file name and CN, a boolean `revoked`), unique names that
   do not clash with `ca` or `tls-crypt`, the port and protocol, and that
   `openvpn_pki_remote` is not the placeholder once there is a profile to
   write. Types and choices are checked by
   [`meta/argument_specs.yml`](meta/argument_specs.yml).
2. **Creates the missing keys** — ECDSA P-256 for the CA, the server and every
   client that is not revoked, and the tls-crypt key. An existing key is never
   replaced, encrypted or not.
3. **Issues the certificates**: a self-signed CA, the server's (`serverAuth`)
   and the clients' (`clientAuth`), each issued again only when its subject,
   key or CA changes.
4. **Writes the CRL** with the certificates of the revoked clients. It changes
   only when the list of revoked clients does.
5. **Writes a profile per client** with everything inline (CA, certificate,
   key, tls-crypt key). A profile is compared with the decrypted content of
   the existing one and rewritten only when it differs. Profiles of revoked
   clients are deleted.
6. **Reports the secrets in plain text**, with the `ansible-vault encrypt`
   command for exactly those files.

On the controller it leaves:

```text
pki/
├── ca.crt                  public
├── crl.pem                 public  certificate revocation list
├── issued/<name>.crt       public  the server's and every client's certificate
├── private/                SECRET  encrypt every file in here:
│   ├── ca.key                      the CA key - whoever has it can issue
│   │                               certificates the server lets in
│   ├── server.key                  the server's key
│   ├── tls-crypt.key               shared by the server and all clients;
│   │                               without it nobody even gets a TLS handshake
│   └── <client>.key                each client's key
└── clients/<client>.ovpn   SECRET  encrypt: the profile has the client's key
                                    and the tls-crypt key inline
```

`private/` and `clients/` are `0700`, the keys and profiles `0600`.

## Variables

See [`defaults/main.yml`](defaults/main.yml) for the full commented list.

| Variable                      | Default                  | Purpose |
|-------------------------------|--------------------------|---------|
| `openvpn_pki_dir`             | `{{ playbook_dir }}/pki` | Where the files go, on the controller |
| `openvpn_pki_ca_name`         | `Homelab OpenVPN CA`     | CN of the CA — changing it replaces the CA and every certificate |
| `openvpn_pki_server_name`     | `server`                 | CN and file name of the server's certificate; the profiles only accept this CN |
| `openvpn_pki_ca_days`         | `3650`                   | Validity of the CA certificate and the CRL |
| `openvpn_pki_cert_days`       | `1825`                   | Validity of the server and client certificates |
| `openvpn_pki_remote`          | `vpn.example.com`        | Address the clients connect to — **must be set**, the default is refused |
| `openvpn_pki_port`            | `1194`                   | Must match the server's `openvpn_port` |
| `openvpn_pki_proto`           | `udp`                    | `udp` / `tcp` / `udp6` / `tcp6`, must match the server's `openvpn_proto` |
| `openvpn_pki_clients`         | `[]`                     | `- name: <name>`, optionally `revoked: true` |
| `openvpn_pki_client_template` | `openvpn-client.ovpn.j2` | Template of the profiles |

The validity periods apply when a certificate is issued — changing them
reissues nothing.

## Usage

```yaml
# group_vars/all.yml (or the play's vars)
openvpn_pki_remote: vpn.mydomain.tld
openvpn_pki_clients:
  - name: kirgo-laptop
  - name: kirgo-phone
```

On its own, from the directory with the playbooks (so the files land in
`./pki`):

```yaml
# openvpn-pki.yml
- name: Create the OpenVPN PKI and the client profiles
  hosts: localhost
  connection: local
  gather_facts: false
  roles:
    - role: homelab.openvpn_pki
```

```bash
ansible-playbook openvpn-pki.yml
```

Or in the play that deploys the server — the role runs once on the controller
before the server gets the files:

```yaml
- hosts: vpn
  become: true
  roles:
    - role: homelab.openvpn_pki
    - role: homelab.openvpn
```

```yaml
# group_vars/vpn.yml - for the homelab.openvpn role
openvpn_ca_cert_src: pki/ca.crt
openvpn_server_cert_src: pki/issued/server.crt
openvpn_server_key_src: pki/private/server.key
openvpn_tls_crypt_key_src: pki/private/tls-crypt.key
openvpn_crl_src: pki/crl.pem
```

### Encrypting the secrets

Before committing, encrypt every key and profile. The last task of the role
prints the exact command for whatever is still plain text:

```bash
ansible-vault encrypt pki/private/*.key pki/clients/*.ovpn
```

Encrypted files keep working: this role and the `homelab.openvpn` role read
them through the `file` lookup, which decrypts Vault files on the fly — run
the playbook with `--ask-vault-pass` or a vault password file.

Give a user their profile decrypted, over a secure channel:

```bash
ansible-vault decrypt --output kirgo-laptop.ovpn pki/clients/kirgo-laptop.ovpn
```

### Adding, revoking and renewing

* **Add a client**: add it to `openvpn_pki_clients` and re-run. Its key and
  profile are new, so they are in plain text — encrypt them.
* **Revoke a client**: set `revoked: true` on its entry and re-run. Its
  certificate goes on the CRL and its profile is deleted. Keep the entry: once
  it is removed, the certificate drops off the CRL again. Then redeploy the
  CRL to the server (`--tags openvpn_pki` with the `homelab.openvpn` role in
  the same play runs both sides).
* **Renew a certificate**: delete it (`ca.crt` or `issued/<name>.crt`) and
  re-run. Its key stays, the affected profiles are rewritten. A new `ca.crt`
  means new certificates for everyone and new profiles for all clients.
* **Change the remote, port or protocol**: re-run. Every profile is rewritten
  in plain text and has to be encrypted again.

### Custom profiles

To add directives (e.g. `redirect-gateway def1`, `pull-filter`), copy
[`templates/openvpn-client.ovpn.j2`](templates/openvpn-client.ovpn.j2) into
the playbook's `templates/` directory under a new name and point
`openvpn_pki_client_template` to it. The template sees the role's variables
and the client's entry as `item`.

## Tags

| Tag                    | Tasks |
|------------------------|-------|
| `always`               | Input validation |
| `openvpn_pki`          | Everything below |
| `openvpn_pki_keys`     | Directories, private keys, tls-crypt key |
| `openvpn_pki_certs`    | CA, server and client certificates, CRL |
| `openvpn_pki_profiles` | Client profiles |
| `openvpn_pki_report`   | The plain-text secrets report |

## Notes

* **Existing keys are never touched.** `openssl_privatekey` cannot read a
  Vault-encrypted key and would replace it — for `ca.key` that voids every
  certificate. So the role only creates keys whose file does not exist.
* **Everything else is compared, not overwritten**: certificates are issued
  again only when what they certify changed, the CRL only when the revoked
  list did, and a profile only when its rendered content differs from the
  decrypted existing one. An unchanged encrypted file stays encrypted.
* **Anything the role writes is plain text**, including a profile it had to
  rewrite. The report at the end lists those files on every run.
* **The CRL is valid as long as the CA** (`openvpn_pki_ca_days`): OpenVPN
  turns every client away once the CRL has expired.
* **Runs once per play.** With several hosts in the play, the role uses the
  variables of the first one.
* The tls-crypt key is generated without OpenVPN: 256 random bytes in the
  format of `openvpn --genkey secret`.
* `--diff` never shows the tls-crypt key or the profiles.

## Testing

A Molecule scenario writes the PKI into Molecule's ephemeral directory on the
controller and uses a Debian 13 container to try it out:

* [`prepare.yml`](molecule/default/prepare.yml) runs the role with two
  clients, `alice` and `bob`, and encrypts the secrets with Ansible Vault (a
  test-only password in `molecule/default/vault-password`).
* [`converge.yml`](molecule/default/converge.yml) revokes `bob` and adds
  `carol` on top of the encrypted PKI; the second run must change nothing.
* [`verify.yml`](molecule/default/verify.yml) checks the files and their
  modes, that the encrypted keys and `alice`'s profile were left encrypted,
  the certificates' CNs, key usages and validity, and that the CRL revokes
  exactly `bob`. Then it starts an OpenVPN server from the PKI in the
  container: `alice` and `carol` connect with their profiles, `bob` is turned
  away.

CI runs it, together with both linters, on every pull request:

```bash
python3 -m venv .venv && . .venv/bin/activate
pip install -r requirements-dev.txt     # same pinned versions as CI
yamllint --strict .
ansible-lint
molecule test
```

ansible-lint and Molecule install the collections from
[`requirements.yml`](requirements.yml) on their own.

## Releases

PRs are squash-merged, so the PR title becomes the commit on `main`. It must be
a [Conventional Commit](https://www.conventionalcommits.org/) (a check enforces
this), because it decides the next version and the changelog entry:

| PR title                                              | Next release | Changelog       |
|-------------------------------------------------------|--------------|-----------------|
| `feat!: …` or a `BREAKING CHANGE:` footer             | major        | ⚠ Breaking      |
| `feat: …`                                             | minor        | Features        |
| `fix:` / `perf:` / `revert:` / `docs: …`              | patch        | own section     |
| `ci:` / `test:` / `refactor:` / `build:` / `style:` / `chore: …` | none | not listed |

After each merge, [release-please](https://github.com/googleapis/release-please)
opens or updates a release PR with the version bump and the new `CHANGELOG.md`
entry. Merging that PR tags the release (e.g. `1.2.0`) and publishes a GitHub
release.

PRs opened with the default `GITHUB_TOKEN` do not trigger workflows, so the
release PR gets no CI checks. If branch protection requires them, add a
fine-grained PAT as the `RELEASE_PLEASE_TOKEN` secret.

## License

MIT
