[![Molecule](https://github.com/iamenr0s/ansible-role-container-hardening/actions/workflows/molecule.yml/badge.svg)](https://github.com/iamenr0s/ansible-role-container-hardening/actions/workflows/molecule.yml) ![Ansible Role](https://img.shields.io/ansible/role/d/iamenr0s/container_hardening) [![CodeFactor](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-container-hardening/badge)](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-container-hardening)

# Ansible Role: Container Hardening

> Standalone copy of `container_hardening` from the [iamenr0s.linux_hardening](https://github.com/iamenr0s/linux-hardening-collection) collection. Variables keep the collection-wide `linux_hardening_*` prefix.

Audit-first host configuration for **Docker Engine**. It owns a few keys of `/etc/docker/daemon.json` and the permissions of the Docker configuration files, and reports the rest. It does not install, remove or upgrade Docker, and it does not harden individual containers or images. Podman, containerd and Kubernetes nodes are out of scope.

Without a `dockerd` binary (`/usr/bin` or `/usr/sbin`) the role says so and does nothing.

## What it checks

| Item | Remediation |
|---|---|
| `daemon.json` managed keys (see below) | merged into the file: our keys get exactly our values, **every other key is kept** |
| `daemon.json` is not valid JSON | **reported only**; the role refuses to overwrite it |
| `/etc/docker` root-owned and not writable by group/others; `daemon.json` root-owned, 0644 or stricter | fixed (a stricter mode is left alone) |
| The running daemon applies the file (live-restore, log driver, `icc`, read back from `docker info` and the default bridge) | restart, **only with `linux_hardening_container_restart: true`**; otherwise reported as not fixable |
| Anything listening on tcp/2375 (unencrypted API) | **reported only** |
| Members of the `docker` group beyond `linux_hardening_container_docker_group_allowed` | **reported only** (root-equivalent; never removed) |

Managed keys, `linux_hardening_container_daemon` (override the whole dictionary to change it; an omitted key is not managed):

| Key | Value | Why |
|---|---|---|
| `icc` | `false` | no unrestricted traffic between containers on the default bridge |
| `no-new-privileges` | `true` | no privilege gain through setuid binaries |
| `live-restore` | `true` | containers keep running while the daemon restarts |
| `userland-proxy` | `false` | no `docker-proxy` processes |
| `log-driver` / `log-opts` | `json-file`, `10m` x 3 | bounded logs |

The selection follows the common daemon-level items of Docker hardening guides, but is **not a verified CIS mapping**. `userns-remap`, TLS and `iptables` are deliberately not managed: they change how existing containers and volumes behave.

## Safety
- **A restart stops every running container** unless `live-restore` is already on in the running daemon. That is why it needs its own switch, and why the role does not restart on its own. Without the switch the file is written, the running daemon keeps its old settings, and the audit says so.
- `icc: false` breaks containers that talk to each other over the default bridge by IP (user-defined networks are not affected). `no-new-privileges` breaks images that rely on setuid binaries.
- The file is validated by `dockerd --validate` (Docker 23+) before it replaces the old one, and the old one is backed up (`linux_hardening_backup`).
- A stopped daemon is left stopped; the role never starts Docker.

## Modes
`audit` (default) is read-only and sets `linux_hardening_container_findings` (`key, expected, actual, fixable`). `remediate` needs `linux_hardening_confirm_remediation: true`; afterwards only fixable findings fail the run.

## Role Variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `linux_hardening_mode` | str | `audit` | audit is read-only. Choices: `audit`, `remediate`. |
| `linux_hardening_confirm_remediation` | bool | `false` | Must be true before remediate changes anything. |
| `linux_hardening_backup` | bool | `true` | Back up daemon.json when it changes. |
| `linux_hardening_verify` | bool | `true` | Re-audit after remediation. |
| `linux_hardening_fail_on_error` | bool | `true` | Fail the play when audit finds deviations. |
| `linux_hardening_container_enabled` | bool | `true` | Run the role. |
| `linux_hardening_container_config` | path | `/etc/docker/daemon.json` | Docker daemon configuration file. |
| `linux_hardening_container_daemon` | dict | see `defaults/main.yml` | daemon.json keys this role owns, with exact values; other keys are kept. |
| `linux_hardening_container_restart` | bool | `false` | Allow restarting dockerd so the configuration takes effect. |
| `linux_hardening_container_docker_group_allowed` | list of str | `[]` | Users allowed in the docker group; others are reported. |

**Idempotent:** yes. A second `remediate` run reports no changes (the Molecule `idempotence` step checks this).

**Check mode:** `remediate` with `--check --diff` shows what would change without writing anything; the post-remediation re-audit is skipped in check mode.

## Example
```yaml
- hosts: docker_hosts
  become: true
  roles:
    - role: iamenr0s.container_hardening
      vars:
        linux_hardening_mode: remediate
        linux_hardening_confirm_remediation: true
        linux_hardening_container_restart: true
        linux_hardening_container_docker_group_allowed: [deploy]
```

## Verification
Container scenarios use a stub `dockerd` (a container cannot run the daemon for this), which is enough for everything the role does to files: `default` (merge keeps unrelated keys, modes fixed, backup made, broken JSON never overwritten, clean second audit), `audit` (no Docker: nothing happens), `refuse`. **VM scenario `vagrant`** (libvirt; `sg libvirt -c "molecule test -s vagrant"`; box `MOLECULE_VAGRANT_BOX`) with a real Docker Engine (docker-ce on EL, `docker.io` on Debian): a weak daemon and a running container, remediation **without** the switch (file changed, running daemon and container untouched, finding reported), remediation **with** it, then the daemon really reports live-restore, log driver and `icc=false`, a new container has `NoNewPrivs: 1` and the log options, a container **survives** a `dockerd` restart, and everything holds after a **reboot**.

## Rollback
Restore `/etc/docker/daemon.json.<timestamp>~`, then restart Docker.

## Known gaps
- `userland-proxy` and `no-new-privileges` have no read-back from the daemon; the container scenarios only prove the file, the VM scenario proves `no-new-privileges`.
- Not tested with a rootless Docker, with `userns-remap`, on Ubuntu or Fedora VMs, or on EL8/EL10.
- The CIS Docker items about images, containers at runtime, Swarm and the audit rules for Docker files are not covered.
- CIS IDs not mapped.

CI & Release (maintainers)
----------------------------
A single workflow (`.github/workflows/molecule.yml`) runs lint and the full Molecule distro matrix on pushes to `main`, PRs, and `v*` tags. On `v*` tags, a `release` job publishes to Ansible Galaxy after all tests pass.

One-time setup:

```bash
# Galaxy publishing key (environment-scoped, get it from galaxy.ansible.com/ui/token)
gh secret set GALAXY_API_KEY --repo iamenr0s/ansible-role-container-hardening

# Code scanning notifications (Slack webhook URL; for Discord append /slack to the webhook URL)
gh secret set SECURITY_ALERT_WEBHOOK --repo iamenr0s/ansible-role-container-hardening
```

`.github/workflows/code-scanning-notify.yml` polls the code-scanning API every 6 hours and posts new or updated open alerts to that webhook (GitHub Actions cannot trigger on `code_scanning_alert` directly).

To release: tag a commit `vX.Y.Z` and push the tag — CI gates the Galaxy publish.

## Contributing

Contributions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) for the
local pipeline commands and pull request checklist. This project follows the
[Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md).

## Security

See [SECURITY.md](SECURITY.md) — GitHub private vulnerability reporting, no
public issues for security bugs.

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history and release notes.

## License

This project is licensed under the [MIT License](LICENSE).

## Author Information

Author: iamenr0s

Galaxy: `iamenr0s.ansible_role_container_hardening`
