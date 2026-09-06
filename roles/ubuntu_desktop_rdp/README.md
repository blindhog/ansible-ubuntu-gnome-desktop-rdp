# ubuntu_desktop_rdp

Installs the full Ubuntu desktop and enables GNOME Remote Desktop's
system-level **Remote Login** (headless RDP) on Ubuntu 24.04–26.04 hosts.
Once applied, the host accepts RDP connections and starts a fresh GNOME
session — no console login or attached monitor required.

## What it does

1. Installs the packages listed in `files/requirements.apt` (`ubuntu-desktop`,
   `gnome-remote-desktop`).
2. Generates a self-signed TLS certificate/key under
   `/var/lib/gnome-remote-desktop/`, owned by the `gnome-remote-desktop`
   service user.
3. Installs a polkit rule so root can configure the system daemon
   non-interactively (see Notes).
4. Configures the system daemon via `grdctl --system` (TLS cert/key, RDP
   credentials) and enables RDP.
5. Enables and starts `gnome-remote-desktop.service`.

## Required variables

| Variable       | Description          |
|----------------|----------------------|
| `rdp_username` | RDP login username   |
| `rdp_password` | RDP login password   |

Set these in `vars/rdp_credentials.yml` (see the repository root).

## Key defaults

See `defaults/main.yml`. Notable values:

| Variable             | Default                                 |
|----------------------|-----------------------------------------|
| `ubuntu_desktop_rdp_tls_dir`        | `/var/lib/gnome-remote-desktop`         |
| `ubuntu_desktop_rdp_tls_cert_days`  | `3650`                                  |
| `ubuntu_desktop_rdp_service_name`   | `gnome-remote-desktop.service`          |

## Notes

- `grdctl --system` calls the gnome-remote-desktop system D-Bus daemon, which
  is gated by the polkit action `org.gnome.remotedesktop.configure-system-daemon`
  (default `auth_admin`). Non-interactively (Ansible, or SSH with no polkit
  agent) that authentication cannot complete, so the role installs
  `/etc/polkit-1/rules.d/49-gnome-remote-desktop-system.rules` granting root a
  passwordless `YES` for that action. Root is already privileged, so this only
  removes the interactive prompt.
- The TLS certificate is **self-signed**, so RDP clients show an
  untrusted-certificate warning on first connect. This is expected for
  internal use; supply your own cert/key paths to avoid it.
- Password idempotency is keyed on the username: the role re-sets
  credentials only when the configured username is not already present in
  `grdctl --system status`. Changing only the password requires clearing the
  stored credential (or temporarily changing the username) so the task runs.
