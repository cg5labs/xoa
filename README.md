# Xen Orchestra Appliance Ansible Playbook

This repository builds a Xen Orchestra Appliance (XO/XOA-style) host with Ansible. The playbook installs Xen Orchestra from source, runs `xo-server` on TCP `8000`, places nginx in front of it on HTTPS TCP `443`, and can create an initial operator user with `xo-cli`.

## What the Playbook Does

- Installs OS packages required to build and run Xen Orchestra.
- Renders the XO server configuration from `roles/xoa/templates/custom.config.toml.j2`.
- Runs the installer script from `roles/xoa/templates/xo_install.sh.j2`.
- Writes the Yarn build log to `/var/log/yarn-build.log` on the target.
- Installs `xo-cli` when `install_xo_cli` is enabled.
- Reboots the target after XO installation.
- Configures nginx as an HTTPS reverse proxy to `http://127.0.0.1:8000`.
- Opens SSH and HTTPS with UFW.
- Optionally creates an injected admin/operator user and removes the default admin user.

## Local Test With Vagrant

The `Vagrantfile` creates a Debian Bookworm VM with the `libvirt` provider and runs `playbooks/xoa.yaml`.

### Requirements

- Vagrant
- `vagrant-libvirt`
- libvirt/KVM
- Ansible available to Vagrant
- OpenSSL on the host

### Run

From the repository root:

```bash
vagrant up
```

The Vagrant flow generates self-signed nginx TLS materials in:

```text
roles/nginx/files/nginx.key
roles/nginx/files/nginx.crt
```

The TLS generation step is idempotent: it does not recreate the key/certificate if both files already exist and are non-empty.

### Access

Vagrant forwards guest HTTPS TCP `443` to host TCP `8443`:

```text
https://127.0.0.1:8443
```

Because the certificate is self-signed, use `-k` with curl or accept the browser warning:

```bash
curl -kI https://127.0.0.1:8443
```

For `vagrant-libvirt`, forwarded ports are implemented as host-side SSH tunnels. The guest reboot during provisioning can drop that tunnel, so the `Vagrantfile` includes a libvirt-only post-`up` trigger that restores the `127.0.0.1:8443 -> guest 443` tunnel. If host access stops working while the VM is running, run:

```bash
vagrant up
```

### Monitor Build Progress

During provisioning, the XO build log is on the guest:

```bash
vagrant ssh -c 'sudo tail -f /var/log/yarn-build.log'
```

## Homelab Installation With Ansible

Use this flow for a real target host, for example `xoa.internal`.

### Requirements

- Debian/Ubuntu-like target with SSH access.
- A privileged SSH user that can use `sudo`.
- DNS or `/etc/hosts` resolves `xoa.internal` to the target.
- TLS certificate and key available on the Ansible control machine.

Place your TLS files where the nginx role can copy them from. By default, the role expects:

```text
roles/nginx/files/nginx.crt
roles/nginx/files/nginx.key
```

For production-like homelab use, replace the Vagrant self-signed files with your own certificate for `xoa.internal`.

### Inventory Example

Create an inventory file, for example `inventory.ini`:

```ini
[xoa]
xoa.internal ansible_user=debian
```

The playbook currently targets the Ansible host group `default`, so either use an inventory alias:

```ini
[default]
xoa.internal ansible_user=debian
```

or change `hosts: default` in `playbooks/xoa.yaml` to your preferred group name, such as `hosts: xoa`.

### Run the Playbook

Inject the admin/operator username and password at runtime:

```bash
ansible-playbook -i inventory.ini playbooks/xoa.yaml \
  -e ansible_user=debian \
  -e nginx_server_name=xoa.internal \
  -e xo_cli_email=admin@xoa.internal \
  -e xo_cli_password='change-me'
```

After completion, open:

```text
https://xoa.internal
```

### Recommended Runtime Variables

Common variables to override with `-e`:

```bash
-e nginx_server_name=xoa.internal
-e xo_cli_email=admin@xoa.internal
-e xo_cli_password='change-me'
```

TLS source files can also be overridden:

```bash
-e nginx_tls_certificate_src=my-xoa.crt
-e nginx_tls_key_src=my-xoa.key
```

When overriding TLS source names, place the files in `roles/nginx/files/` unless you also update the role pathing.

## Key Variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `xo_server_port` | `8000` | Local port used by `xo-server`. |
| `nginx_enable_proxy` | `true` | Enables nginx reverse proxy configuration. |
| `nginx_listen_port` | `443` | HTTPS port nginx listens on. |
| `nginx_proxy_backend_url` | `http://127.0.0.1:8000` | Backend XO server URL. |
| `nginx_server_name` | `xsrv011.cg5labs.net` | nginx `server_name`; override for your hostname. |
| `nginx_tls_certificate_src` | `nginx.crt` | Certificate file copied from `roles/nginx/files/`. |
| `nginx_tls_key_src` | `nginx.key` | Private key copied from `roles/nginx/files/`. |
| `install_xo_cli` | `true` | Installs `xo-cli` and enables user setup tasks. |
| `xo_cli_email` | unset | Runtime-injected operator/admin email. |
| `xo_cli_password` | unset | Runtime-injected operator/admin password. |
| `xo_cli_cleanup_default_admin` | `true` | Deletes the default `admin@admin.net` user after creating your user. |

## Notes

- The target is rebooted during provisioning.
- The build can take several minutes.
- Do not commit real private keys or production secrets.
