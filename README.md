# Ansible Nginx Deployment on AWS EC2

Automates the deployment of an Nginx web server on an AWS EC2 instance using
Ansible, with a custom HTML landing page.

## Architecture

```
Local PC ──SSH (.pem)──► Control node (Ubuntu 24.04 EC2, Ansible)
                               │  SSH (ed25519 key, passwordless)
                               ▼
                         Web node (Ubuntu 24.04 EC2, Nginx on port 80)
```

| Component | Details |
|-----------|---------|
| Control node | Ubuntu 24.04 LTS EC2, Ansible installed via `apt` |
| Web node | Ubuntu 24.04 LTS EC2 running Nginx |
| Security groups | Control: SSH (22) from my IP. Web: SSH (22) from my IP and the control node, HTTP (80) from anywhere |
| Authentication | Key-based SSH only; no passwords |

## Repository Contents

```
.
├── ansible.cfg                  # defaults: inventory, roles path, vault password file
├── inventory.ini                # web node public IP
├── site.yml                     # thin playbook that applies the nginx role
├── group_vars/webservers/
│   ├── vars.yml                 # environment settings (paths, user, port, content)
│   └── vault.yml                # ENCRYPTED secrets (Ansible Vault)
├── roles/nginx/
│   ├── defaults/main.yml        # overridable defaults
│   ├── vars/main.yml            # internal constants (package, service, paths)
│   ├── tasks/main.yml           # install, configure, start, verify
│   ├── handlers/main.yml        # validate config, then restart Nginx
│   ├── templates/               # nginx site config, htpasswd, index.html
│   └── meta/main.yml
├── screenshots/
└── README.md
```

## Infrastructure Setup

1. **Key pair:** created `ansible-lab-key` in the EC2 console, with permissions
   set to `400` on the local machine.
2. **Security groups:** `ansible-control-sg` and `ansible-web-sg` created as
   described above. Port 80 is open so the page is publicly reachable.
3. **Instances:** two t2.micro Ubuntu 24.04 instances launched with the key pair.
4. **Ansible installed** on the control node:
   `sudo apt update && sudo apt install -y ansible`
5. **Passwordless SSH:** generated an ed25519 key on the control node
   (`ssh-keygen -t ed25519`) and appended its public key to
   `~/.ssh/authorized_keys` on the web node.
6. **Inventory:** `inventory.ini` lists the web node's public IP, SSH user
   and private key path.

## Playbook and Role

`site.yml` applies the reusable `nginx` role to the `webservers` group.

| Task | Module | Purpose |
|------|--------|---------|
| Assert secrets exist | `assert` | Fail early if the vault variable is missing |
| Update packages | `apt` | Refresh cache and upgrade, with retries for dpkg lock |
| Install Nginx | `apt` | Install the web server |
| Deploy config, htpasswd, HTML | `template` | Render Jinja2 templates; each change notifies the handler |
| Start and enable | `systemd` | Run now and on boot |
| Verify | `uri` | 200 on `/`, 401 without credentials and 200 with credentials on `/nginx_status` |

**Handlers:** changes to the config, htpasswd or HTML notify the `Restart nginx`
topic. The configuration is validated with `nginx -t` first and Nginx restarts
only if validation passes. Unchanged runs trigger no restart.

**Error handling:** tasks are wrapped in `block/rescue`, apt tasks use
`retries/until`, and secret-handling tasks use `no_log`.

The playbook is idempotent: a second run reports `changed=0`.

## Variables

| Variable | Defined in | Default | Purpose |
|----------|-----------|---------|---------|
| `nginx_web_root` | defaults, group_vars | `/var/www/html` | Document root |
| `nginx_user` | defaults, group_vars | `www-data` | Owner of web files |
| `nginx_port` | defaults, group_vars | `80` | Listen port |
| `nginx_server_name` | defaults | `_` | Server name |
| `site_title`, `site_message` | defaults, group_vars | see file | Page content |
| `nginx_admin_user` | defaults, group_vars | `admin` | Status page user |
| `nginx_admin_password` | group_vars → vault | none | Status page password |
| `ansible_user` | group_vars → vault | none | SSH user |

Precedence: role `defaults` < `group_vars` < command-line `-e`.

## Secrets with Ansible Vault

Sensitive values live in `group_vars/webservers/vault.yml`, encrypted with
AES-256. Plain variables reference them (for example
`nginx_admin_password: "{{ vault_nginx_admin_password }}"`).

```bash
ansible-vault edit group_vars/webservers/vault.yml    # edit secrets
ansible-vault view group_vars/webservers/vault.yml    # read secrets
ansible-playbook site.yml --ask-vault-pass            # or use .vault_pass
```

`.vault_pass` is git-ignored. Assessment vault password: `<throwaway-password>`
(for demonstration only).

## How to Run

```bash
ansible all -m ping                        # test connectivity
ansible-playbook site.yml --syntax-check   # validate syntax
ansible-playbook site.yml                  # deploy
```

## Verification

Open `http://3.90.12.161/` in a browser: 
However the above will be unable as EC2 instance will be deleted.

![Deployed page](screenshots/Deployment_success.jpg)

## Troubleshooting

>

| Problem | Cause | Fix |
|---------|-------|-----|
| `Permissions 0644 for key are too open` | Key file too permissive (common on Windows/OneDrive) | Move the key to `~/.ssh` and run `chmod 400` |
| `Connection timed out` on SSH | Security group missing port 22 or wrong source IP | Update the inbound rule; my public IP may have changed |
| Control node cannot SSH to the web node | Web SG allowed the control node's private SG, but traffic used the public IP | Allowed the control node's public IP (`/32`) on port 22 |
| `Permission denied (publickey)` | Public key not in the target's `authorized_keys` | Re-append the key; check the `ansible_user` is `ubuntu` |
| `Could not get lock /var/lib/dpkg/lock-frontend` | `unattended-upgrades` running right after boot | Wait a few minutes and re-run |
| Browser cannot load the page | Port 80 closed, or browser forced HTTPS | Check the security group and type `http://` explicitly |
| Page works then stops after restart | Public IP changed when the instance was stopped | Update `inventory.ini` or attach an Elastic IP |

## Cleanup
Terminate both EC2 instances, delete the security groups and the key pair to
avoid charges.