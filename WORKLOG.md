# Worklog

## Session 1 — 2026-09-06 (~1 hr)

- Created the `ubuntu_desktop_rdp` role: installs the full Ubuntu desktop and
  enables GNOME Remote Desktop's system-level Remote Login (headless RDP) on
  Ubuntu 24.04–26.04.
  - `defaults/main.yml`: service name, service user, TLS cert paths/lifetime,
    requirements file location.
  - `files/requirements.apt`: `ubuntu-desktop`, `gnome-remote-desktop`.
  - `tasks/main.yml`: validate creds + platform, install packages, generate a
    self-signed TLS cert, configure the daemon via `grdctl --system`, enable
    and start the service.
  - `handlers/main.yml`: restart `gnome-remote-desktop.service` on config change.
  - `README.md`: role documentation.
- Added `install-ubuntu-desktop-rdp.yml` (repo root) that applies the role to
  the `desktop_servers` group with `become`.
- Added `vars/rdp_credentials.yml` with placeholder `rdp_username`/`rdp_password`.
- Expanded the repository `README.md` with layout and usage.
- Added `ansible.cfg` (default inventory + `roles_path`) and
  `inventory/hosts.yml` with `localhost` in the `desktop_servers` group
  (connection var in `inventory/host_vars/localhost/vars.yml`); moved the
  playbook into `playbooks/` and fixed its `vars_files` path.

- Discovered (via a live `grdctl --system` run) that configuring the system
  daemon is gated by the polkit action
  `org.gnome.remotedesktop.configure-system-daemon` (`auth_admin`), which fails
  non-interactively under Ansible/SSH. Added a polkit rule
  (`files/49-gnome-remote-desktop-system.rules`) granting root a passwordless
  `YES`, deployed before the `grdctl` tasks.
- Renamed all role variables to the `ubuntu_desktop_rdp_` prefix to satisfy
  `ansible-lint` (`var-naming[no-role-prefix]`); repo now passes ansible-lint's
  `production` profile and `--syntax-check`.

### Milestones

- First functional role and playbook for the repository.

### Follow-ups

- Smoke-test the `grdctl --system` task sequence against a live Ubuntu 24.04+
  host; the exact `grdctl` subcommand syntax shifted between GNOME 46 and 48.

## Summary

| Metric        | Value |
|---------------|-------|
| Calendar days | 1     |
| Sessions      | 1     |
| Est. hours    | 1     |
