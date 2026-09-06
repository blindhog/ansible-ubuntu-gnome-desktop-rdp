# ansible-ubuntu-gnome-desktop-rdp
Ansible automation to install Ubuntu Desktop and enable RDP access

Enables GNOME Remote Desktop's system-level **Remote Login** (headless RDP)
on Ubuntu 24.04–26.04 servers, so you can RDP into a monitorless host and get
a fresh GNOME session.

## Layout

```
ansible.cfg                            # default inventory + roles_path
inventory/hosts.yml                    # desktop_servers group (localhost)
playbooks/install-ubuntu-desktop-rdp.yml   # playbook that applies the role
roles/ubuntu_desktop_rdp/              # the role (see its README for details)
vars/rdp_credentials.yml               # RDP username/password
```

## Usage

1. Set the RDP credentials:

   ```yaml
   # vars/rdp_credentials.yml
   rdp_username: "myuser"
   rdp_password: "a-strong-password"
   ```

   Optionally encrypt the file: `ansible-vault encrypt vars/rdp_credentials.yml`.

2. Run the playbook. The bundled `inventory/hosts.yml` puts `localhost` in the
   `desktop_servers` group, so it applies to the local machine by default —
   add your own hosts to that group to target servers:

   ```bash
   ansible-playbook playbooks/install-ubuntu-desktop-rdp.yml
   ```

   Add `--ask-vault-pass` if you encrypted the credentials file.

3. Connect with any RDP client on port `3389`, using the credentials above.
   The self-signed TLS certificate triggers an untrusted-certificate warning
   on first connect — expected for internal use.

See [`roles/ubuntu_desktop_rdp/README.md`](roles/ubuntu_desktop_rdp/README.md)
for what the role does and its variables.
