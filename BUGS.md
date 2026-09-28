# Known Bugs

## 1. `grdctl set-credentials` breaks on passwords containing spaces

- **Location:** `roles/ubuntu_desktop_rdp/tasks/main.yml:127-133`
- **Problem:** The password is interpolated bare into the command string:
  `grdctl --system rdp set-credentials {{ rdp_username }} {{ rdp_password }}`.
  `ansible.builtin.command` splits on whitespace, so a password containing a
  space is passed as extra argv entries and the credential is set incorrectly
  (or the command fails). Shell metacharacters are safe since no shell is used.
- **Impact:** Silent wrong-password configuration; RDP login then fails with no
  obvious cause, and `no_log: true` hides the evidence.
- **Fix sketch:** Use `argv:` instead of `cmd:`.
- **Found:** Code review during first playbook run, 2026-09-24.

## 2. Credential idempotency check can false-negative

- **Location:** `roles/ubuntu_desktop_rdp/tasks/main.yml:130`
- **Problem:** `when: rdp_username not in ubuntu_desktop_rdp_status.stdout`
  does a substring match over the whole `grdctl status` output. A username that
  happens to appear elsewhere in that output (e.g. inside a TLS path) makes the
  task skip even when credentials were never set. Conversely the password is
  never compared, so a changed password alone is never reapplied.
- **Impact:** Password changes silently do not take effect on re-runs.
- **Found:** Code review during first playbook run, 2026-09-24.
