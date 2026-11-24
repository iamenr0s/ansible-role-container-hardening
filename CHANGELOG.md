# Changelog

## Unreleased
- Internal: the managed keys that differ come from one filter chain, and the fail decision is two named tasks.
- Change: dropped the `linux_hardening_mode` assert that repeated `meta/argument_specs.yml` (which already rejects other values).
- Fix: `--check` dropped the stale-daemon and tcp/2375 findings (read-only probes were skipped), and the post-remediation re-audit failed; both fixed.
- Fix: `meta/argument_specs.yml` now has the default for `linux_hardening_container_daemon`.
- Change: task names in sub-task files are prefixed.
- Docs: README adds a Role Variables table, the example uses the standalone role name with `become: true`, idempotency and check-mode notes.
- Test: the default Molecule scenario runs a `check` (dry-run remediate) step first.
- CI: every platform in `meta/main.yml` gates the release (no `continue-on-error` legs); lint runs the pinned pre-commit hooks; pre-commit adds gitleaks; actions are pinned by commit SHA; the release uses `ansible-galaxy role import` like the sibling roles.

## 1.0.0
- Split out of the iamenr0s.linux_hardening collection; no functional change.
