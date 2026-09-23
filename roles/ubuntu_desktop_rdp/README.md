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
5. Adds each user in `desktop_users` to the `render` and `video`
   groups so their RDP sessions can access the GPU (see Notes).
6. Enables and starts `gnome-remote-desktop.service`.

## Required variables

| Variable        | Description                                                      |
|-----------------|------------------------------------------------------------------|
| `rdp_username`  | RDP login username                                               |
| `rdp_password`  | RDP login password                                               |
| `desktop_users` | Local Linux accounts added to the `render` and `video` groups    |

Set `rdp_username` and `rdp_password` in `vars/rdp_credentials.yml` (see the
repository root). Set `desktop_users` per host in
`inventory/host_vars/<host>/vars.yml`.

## Key defaults

See `defaults/main.yml`. Notable values:

| Variable             | Default                                 |
|----------------------|-----------------------------------------|
| `ubuntu_desktop_rdp_tls_dir`        | `/var/lib/gnome-remote-desktop`         |
| `ubuntu_desktop_rdp_tls_cert_days`  | `3650`                                  |
| `ubuntu_desktop_rdp_service_name`   | `gnome-remote-desktop.service`          |

## Which account is which

`desktop_users`, `ubuntu_desktop_rdp_service_user`, and `rdp_username` are
three separate identities. They are not interchangeable:

| Variable                          | What it is                                                                                              | What the role does with it                                                                |
|-----------------------------------|---------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------|
| `rdp_username`                    | An RDP-only credential stored by the daemon. It is not a Linux account and need not exist in `/etc/passwd` | Passes it to `grdctl --system rdp set-credentials`                                      |
| `ubuntu_desktop_rdp_service_user` | The `gnome-remote-desktop` system account the RDP daemon runs as (created by the package, no login)    | Makes it the owner of the TLS certificate and key                                         |
| `desktop_users`                   | Real people: the local Linux accounts you sign in as at the GNOME login screen after connecting        | Adds each one to `render` and `video` so their GNOME session can use the GPU (`/dev/dri`) |

Connecting is a two-step login: the RDP client authenticates to the daemon
with `rdp_username`/`rdp_password`, then you sign in at the GNOME login screen
as one of the `desktop_users` with that account's own Linux password.

Put your own login accounts in `desktop_users`, usually in
`inventory/host_vars/<host>/vars.yml`. Leave `ubuntu_desktop_rdp_service_user`
at its default unless your distribution runs the daemon under a different
account. Never add the service user to `desktop_users`.

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
- RDP sessions need read/write access to the GPU device nodes under
  `/dev/dri`, which are owned by the `render` and `video` groups. List the
  local users who log in over RDP in `desktop_users`; the role appends those
  groups (equivalent to `usermod -aG render,video <user>`) and fails if a
  listed user does not exist. Membership applies at the user's next
  login, so sign out of any existing session afterwards.
