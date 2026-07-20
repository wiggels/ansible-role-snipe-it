# Ansible Role: Snipe-IT

[![CI](https://github.com/wiggels/ansible-role-snipe-it/actions/workflows/ci.yml/badge.svg)](https://github.com/wiggels/ansible-role-snipe-it/actions/workflows/ci.yml)
[![Ansible Galaxy](https://img.shields.io/badge/galaxy-wiggels.snipeit-blue)](https://galaxy.ansible.com/wiggels/snipeit)

Installs, configures, and upgrades a complete
[Snipe-IT](https://snipeitapp.com/) asset management server on Ubuntu — NGINX,
PHP-FPM, MySQL, composer dependencies, database migrations, and the Laravel
scheduler cron, all in one role.

## Requirements

- Ubuntu 24.04 (noble) or 26.04 (resolute)
- ansible-core 2.16 or newer on the control node
- The `ansible.mysql` collection

```sh
ansible-galaxy role install wiggels.snipeit
ansible-galaxy collection install ansible.mysql
```

PHP is installed from [`ppa:ondrej/php`](https://launchpad.net/~ondrej/+archive/ubuntu/php)
(8.4) on Ubuntu 24.04. Ubuntu 26.04 ships PHP 8.5 natively, so the PPA is
skipped there automatically.

> [!NOTE]
> The `requirements.yml` in this repository also lists `community.docker` and
> `ansible.posix` — those are only needed to run the Molecule test suite, not
> to use the role.

## Role Variables

All variables (with documentation) live in `defaults/main.yml`. The most
important ones:

### Install

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `snipe_install_version` | `8.6.3` | Snipe-IT release to install or upgrade to (git tag without the `v`) |
| `snipe_install_dir` | `/opt/snipe-it` | Install location |
| `snipe_repo_url` | `https://github.com/grokability/snipe-it.git` | Source repository |
| `snipe_php_ver` | `8.4` on 24.04, `8.5` on 26.04 | PHP version to install |
| `snipe_php_ppa_enabled` | `true` on 24.04, `false` on 26.04 | Whether to add `ppa:ondrej/php` |
| `snipe_domain` | `localhost` | Server name for the NGINX vhost and app URL |
| `snipe_cleanup` | `false` | Purge existing apache/php/nginx packages before installing |
| `snipe_scheduler_enabled` | `true` | Install the Laravel scheduler cron for `www-data` |

### SSL

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `snipe_ssl_enabled` | `false` | Serve over HTTPS (you provide the cert and key) |
| `snipe_ssl_cert` | `/etc/ssl/certs/{{ snipe_domain }}-fullchain.crt` | Certificate path |
| `snipe_ssl_key` | `/etc/ssl/private/{{ snipe_domain }}.key` | Private key path |

### Database

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `snipe_db_host` | `localhost` | MySQL host — when `localhost`, this role installs and configures MySQL locally |
| `snipe_db_database` | `snipeit` | Database name |
| `snipe_db_username` | `snipeit` | Application DB user |
| `snipe_db_password` | `ChangeMe` | Application DB password — **change this** |
| `snipe_db_admin_user` | unset | Admin user for creating the DB and app user; leave unset to connect as root over the unix socket |
| `snipe_db_admin_password` | unset | Password for the admin user |
| `snipe_db_password_salt` | derived | 20-character salt for the app user's `caching_sha2_password` hash (keeps password management idempotent) |

### Application

| Variable | Default | Description |
| -------- | ------- | ----------- |
| `snipe_app_key` | `ChangeMe` | Laravel app key — auto-generated on first install and preserved on later runs |
| `snipe_app_timezone` | `America/New_York` | App timezone |
| `snipe_app_locale` | `en-US` | App locale |
| `snipe_mail_host` / `snipe_mail_port` | `localhost` / `25` | Outgoing mail server |
| `snipe_mail_username` / `snipe_mail_password` | `false` | SMTP auth credentials (omitted from `.env` while `false`) |
| `snipe_mail_from_addr` | `snipe-it@example.com` | From address |
| `snipe_env_extra` | `{}` | Dict of additional `.env` settings rendered verbatim (e.g. `ENABLE_HSTS: "true"`) |

See `defaults/main.yml` for the full list (sessions, cookies, login
throttling, backup permissions, etc.).

## Example Playbook

```yaml
- hosts: all
  tasks:
    - name: Install and Configure Snipe-IT
      ansible.builtin.include_role:
        name: wiggels.snipeit
      vars:
        snipe_domain: snipeit.example.com
        snipe_db_password: "{{ vault_snipe_db_password }}"
        snipe_mail_host: smtp.example.com
        snipe_mail_from_addr: snipe-it@example.com
```

Upgrading an existing install is the same play with a newer
`snipe_install_version` — the role preserves the existing `APP_KEY`, checks
out the new tag, reinstalls composer dependencies, and runs migrations.

## Upgrading from v2 of this role

v3 modernized the role for Snipe-IT 8.x and dropped some legacy behavior:

- Renamed: `snipe_mail_driver` → `snipe_mail_mailer` (Snipe-IT 8.x `.env` format)
- Removed: `snipe_mail_encryption`, `snipe_mail_auto_embed` (no longer supported
  upstream); added `snipe_mail_tls_verify_peer`
- Defaults changed: `snipe_app_locale` is now `en-US`, `snipe_session_lifetime`
  is `12000`, and `snipe_api_token_expiration_years` is `15` (upstream defaults)
- The `nginxinc.nginx_core` collection is no longer used — NGINX now comes from
  Ubuntu's repos and the vhost is managed directly by this role
- The application DB user now authenticates with `caching_sha2_password`
  (MySQL 8.4+ removed `mysql_native_password`); existing users are migrated on
  the next run
- Ubuntu 20.04/22.04 support was dropped

## Testing

Tests run with [Molecule](https://ansible.readthedocs.io/projects/molecule/)
in Docker (systemd-enabled containers) and cover a full install, an
idempotence check, and a verify stage that confirms the services are running
and the app answers over HTTP:

```sh
pip install ansible-core molecule "molecule-plugins[docker]" docker
ansible-galaxy collection install -r requirements.yml
molecule test                              # Ubuntu 24.04 (default)
MOLECULE_DISTRO=ubuntu2604 molecule test   # Ubuntu 26.04
```

If your Docker socket lives somewhere non-standard (Rancher Desktop, Colima,
etc.), point molecule at it first, e.g.
`export DOCKER_HOST=unix://$HOME/.rd/docker.sock`.

CI runs ansible-lint plus the full Molecule matrix on every push and pull
request, and monthly to catch breakage from new upstream releases.

## Releases

Merges to `main` are tagged automatically once CI passes, with the version
bump derived from commit messages since the last tag:

| Commit message prefix | Bump |
| --------------------- | ---- |
| `breaking: ...`, `feat!: ...`, or a `BREAKING CHANGE:` footer | major |
| `feat: ...` | minor |
| `fix: ...` | patch |

Commits that match none of these (e.g. `docs:`, `chore:`) do not produce a
release.

## License

MIT

## Author Information

Hunter Wigelsworth
[GitHub](https://github.com/wiggels)
[LinkedIn](https://www.linkedin.com/in/wiggels/)
