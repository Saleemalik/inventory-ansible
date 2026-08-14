# Inventory System Deployment Automation with Ansible

Automated deployment of a full-stack **Django + React Inventory System**
on **Ubuntu 24.04 LTS** using **Ansible**.

This project provisions PostgreSQL, Django, React/Vite, Gunicorn, Nginx,
UFW, GitHub SSH authentication, and application log rotation using
Infrastructure as Code (IaC) and idempotent role-based automation.

Application repository:

**https://github.com/Saleemalik/inventory_system**

---

## Features

- Automated Ubuntu server provisioning
- PostgreSQL installation and database configuration
- Django backend deployment
- React/Vite frontend deployment
- Python virtual environment
- Node.js 22 LTS and npm
- Gunicorn systemd service
- Nginx reverse proxy
- GitHub SSH authentication
- Ansible Vault for secrets
- Django migrations and superuser creation
- Static and media configuration
- UFW firewall
- Application log rotation
- Modular and idempotent Ansible roles
- Local VM and AWS EC2 deployment support

---

## Architecture

```text
                    Ansible Controller
                           |
                           | SSH
                           v
                  Ubuntu Managed Server
                           |
                          UFW
                           |
                         Nginx
                       /       \
                      /         \
                 React UI     Gunicorn
                                 |
                               Django
                                 |
                            PostgreSQL
```

Nginx handles:

```text
/          -> React production build
/api/      -> Django/Gunicorn
/admin/    -> Django/Gunicorn
/static/   -> Static files
/media/    -> Media files
```

Gunicorn is not exposed directly to the Internet. Nginx communicates
with Gunicorn through a Unix socket.

---

## Repository Structure

```text
inventory-ansible/
├── ansible.cfg
├── ansible.cfg.example
├── inventory
├── inventory.example
├── requirements.yml
├── README.md
│
├── group_vars/
│   └── all/
│       ├── vars.yml
│       └── vault.yml
│
├── host_vars/
│
├── playbooks/
│   ├── deploy_inventory.yml
│   ├── github_ssh.yml
│   ├── test_backend.yml
│   ├── test_django.yml
│   ├── test_frontend.yml
│   ├── test_gunicorn.yml
│   ├── test_nginx.yml
│   ├── test_ufw.yml
│   └── test_logrotate.yml
│
└── roles/
    ├── common/
    ├── postgresql/
    ├── backend/
    ├── django/
    ├── frontend/
    ├── gunicorn/
    ├── nginx/
    ├── ufw/
    └── logrotate/
```

---

## Roles

| Role | Purpose |
|---|---|
| `common` | System packages and prerequisites |
| `postgresql` | PostgreSQL installation and database |
| `backend` | Git repository, Python environment and dependencies |
| `django` | `.env`, migrations, static/media and superuser |
| `frontend` | Node.js, npm and React production build |
| `gunicorn` | Django systemd service |
| `nginx` | Frontend serving and reverse proxy |
| `ufw` | Firewall configuration |
| `logrotate` | Application log rotation |

`github_ssh.yml` is a separate playbook used to configure GitHub SSH
authentication for the deployment/remote instance.

---

## Requirements

### Ansible Controller

- Ubuntu/Linux
- Python 3
- Ansible
- Git
- OpenSSH client

Install:

```bash
sudo apt update
sudo apt install -y ansible git openssh-client
```

Verify:

```bash
ansible --version
```

### Managed Server

- Ubuntu 24.04 LTS
- SSH access
- Sudo privileges
- Python 3
- Internet access

---

## Installation

Clone the repository:

```bash
git clone https://github.com/Saleemalik/inventory-ansible.git
cd inventory-ansible
```

Install required Ansible collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

---

## Ansible Configuration

Copy the example:

```bash
cp ansible.cfg.example ansible.cfg
```

Example:

```ini
[defaults]
inventory = inventory
roles_path = roles
interpreter_python = auto_silent

[ssh_connection]
pipelining = True
```

Keep machine-specific configuration in the local `ansible.cfg`.
Do not commit private paths or environment-specific configuration.

---

## Inventory Configuration

Copy the example:

```bash
cp inventory.example inventory
```

The example inventory intentionally contains **no real IP addresses**.

Example local server:

```ini
[inventory_servers]
local_inventory ansible_user=maliks app_user=inventory2
```

Example AWS EC2:

```ini
[aws_inventory_servers]
aws_inventory ansible_user=ubuntu ansible_ssh_private_key_file=<PATH_TO_AWS_PRIVATE_KEY> ansible_python_interpreter=/usr/bin/python3.12 app_user=inventory2
```

### System Users

The SSH/deployment user depends on the server:

```text
Local Ubuntu VM  -> maliks
AWS Ubuntu EC2   -> ubuntu
```

### Application User

The application runs as the dedicated unprivileged user:

```text
inventory2
```

The application user is different from the SSH/deployment user.

For a real deployment, add the actual server address in your private
`inventory` file using `ansible_host`.

---

## Test Connectivity

After configuring the real `inventory`:

```bash
ansible all -m ping
```

Or by group:

```bash
ansible inventory_servers -m ping
ansible aws_inventory_servers -m ping
```

The repository does not store real server addresses.

---

## Application Variables

Configure:

```text
group_vars/all/vars.yml
```

Example:

```yaml
app_name: inventory-system

app_user: inventory2
app_group: inventory2

app_directory: /opt/inventory-system-2
venv_directory: /opt/inventory-system-2/venv

deploy_user: "{{ ansible_user }}"

app_repo: "git@github.com:Saleemalik/inventory_system.git"
app_branch: main

postgresql_db: inventory_db
postgresql_user: inventory_user
postgresql_host: localhost
postgresql_port: 5432

static_root: /opt/inventory-system-2/staticfiles
media_root: /opt/inventory-system-2/media

django_debug: "False"
```

---

## Ansible Vault

Create the encrypted secrets file:

```bash
ansible-vault create group_vars/all/vault.yml
```

Example:

```yaml
vault_django_secret_key: "CHANGE_ME"
vault_postgresql_password: "CHANGE_ME"
vault_django_superuser_password: "CHANGE_ME"
```

Edit later:

```bash
ansible-vault edit group_vars/all/vault.yml
```

Never commit plaintext production secrets.

---

## GitHub SSH Authentication

GitHub authentication is handled separately from the main deployment.

Run:

```bash
ansible-playbook playbooks/github_ssh.yml --ask-vault-pass
```

The playbook:

- Creates the SSH directory
- Generates the GitHub deployment key
- Configures `known_hosts`
- Configures SSH for GitHub
- Verifies GitHub authentication

Add the generated public key to the application repository as a
**GitHub Deploy Key**.

Repository:

**https://github.com/Saleemalik/inventory_system**

Verify:

```bash
ssh -T git@github.com
```

---

## Main Deployment

The main playbook is:

```text
playbooks/deploy_inventory.yml
```

Deployment order:

```text
common
   ↓
postgresql
   ↓
backend
   ↓
django
   ↓
frontend
   ↓
gunicorn
   ↓
nginx
   ↓
ufw
   ↓
logrotate
```

Run:

```bash
ansible-playbook playbooks/deploy_inventory.yml --ask-vault-pass
```

For a specific group:

```bash
ansible-playbook playbooks/deploy_inventory.yml \
  --ask-vault-pass \
  --limit aws_inventory_servers
```

---

## Main Playbook

```yaml
---
- name: Deploy Inventory System
  hosts: inventory_servers
  become: true

  roles:
    - common
    - postgresql
    - backend
    - django
    - frontend
    - gunicorn
    - nginx
    - ufw
    - logrotate
```

---

## Django Configuration

The Django role configures:

```text
SECRET_KEY
DEBUG
DB_NAME
DB_USER
DB_PASSWORD
DB_HOST
DB_PORT
ALLOWED_HOSTS
STATIC_ROOT
MEDIA_ROOT
```

It also runs:

```yaml
- name: Run Django system checks
  become_user: "{{ app_user }}"
  ansible.builtin.command:
    cmd: "{{ django_python }} {{ django_manage_py }} check"

- name: Run migrations
  become_user: "{{ app_user }}"
  ansible.builtin.command:
    cmd: "{{ django_python }} {{ django_manage_py }} migrate --noinput"

- name: Collect static files
  become_user: "{{ app_user }}"
  ansible.builtin.command:
    cmd: "{{ django_python }} {{ django_manage_py }} collectstatic --noinput"
```

---

## Frontend

Production API configuration:

```text
VITE_API_BASE=/api
```

The frontend is built with:

```bash
npm install
npm run build
```

Production output:

```text
/opt/inventory-system-2/frontend/dist
```

---

## Gunicorn

Gunicorn runs Django through systemd as `inventory2`.

Example:

```ini
[Service]
User=inventory2
Group=inventory2
WorkingDirectory=/opt/inventory-system-2
ExecStart=/opt/inventory-system-2/venv/bin/gunicorn     --workers 3     --bind unix:/opt/inventory-system-2/gunicorn.sock     inventory_system.wsgi:application
```

Socket:

```text
/opt/inventory-system-2/gunicorn.sock
```

---

## Nginx

Nginx serves the React production build and proxies Django requests.

```nginx
server {
    listen 80;
    server_name _;

    root /opt/inventory-system-2/frontend/dist;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        include proxy_params;
        proxy_pass http://unix:/opt/inventory-system-2/gunicorn.sock;
    }

    location /admin/ {
        include proxy_params;
        proxy_pass http://unix:/opt/inventory-system-2/gunicorn.sock;
    }

    location /static/ {
        alias /opt/inventory-system-2/staticfiles/;
    }

    location /media/ {
        alias /opt/inventory-system-2/media/;
    }
}
```

Validate:

```bash
sudo nginx -t
```

---

## UFW

Required public ports:

```text
22  SSH
80  HTTP
443 HTTPS
```

Gunicorn port `8000` is not publicly exposed.

Example:

```yaml
- name: Allow SSH
  community.general.ufw:
    rule: allow
    port: "22"
    proto: tcp

- name: Allow HTTP
  community.general.ufw:
    rule: allow
    port: "80"
    proto: tcp

- name: Allow HTTPS
  community.general.ufw:
    rule: allow
    port: "443"
    proto: tcp

- name: Enable UFW
  community.general.ufw:
    state: enabled
    policy: deny
```

---

## Logrotate

Application logs:

```text
/opt/inventory-system-2/logs/
```

Configuration:

```text
/etc/logrotate.d/inventory-system
```

Example:

```text
/opt/inventory-system-2/logs/*.log {
    daily
    rotate 7
    missingok
    notifempty
    compress
    delaycompress
    copytruncate
    su inventory2 inventory2
}
```

Test:

```bash
sudo logrotate -vf /etc/logrotate.d/inventory-system
```

---

## Individual Role Testing

```bash
ansible-playbook playbooks/test_backend.yml --ask-vault-pass
ansible-playbook playbooks/test_django.yml --ask-vault-pass
ansible-playbook playbooks/test_frontend.yml --ask-vault-pass
ansible-playbook playbooks/test_gunicorn.yml --ask-vault-pass
ansible-playbook playbooks/test_nginx.yml --ask-vault-pass
ansible-playbook playbooks/test_ufw.yml --ask-vault-pass
ansible-playbook playbooks/test_logrotate.yml --ask-vault-pass
```

---

## Verification

PostgreSQL:

```bash
sudo systemctl status postgresql
```

Django:

```bash
sudo -u inventory2   /opt/inventory-system-2/venv/bin/python   /opt/inventory-system-2/manage.py check
```

Frontend:

```bash
ls -lah /opt/inventory-system-2/frontend/dist
```

Gunicorn:

```bash
sudo systemctl status gunicorn
ls -l /opt/inventory-system-2/gunicorn.sock
```

Nginx:

```bash
sudo nginx -t
sudo systemctl status nginx
```

UFW:

```bash
sudo ufw status verbose
```

Logrotate:

```bash
sudo logrotate -vf /etc/logrotate.d/inventory-system
```

---

## Application Paths

```text
Application:
 /opt/inventory-system-2

Virtual environment:
 /opt/inventory-system-2/venv

Frontend:
 /opt/inventory-system-2/frontend

Frontend build:
 /opt/inventory-system-2/frontend/dist

Static files:
 /opt/inventory-system-2/staticfiles

Media files:
 /opt/inventory-system-2/media

Logs:
 /opt/inventory-system-2/logs

Gunicorn socket:
 /opt/inventory-system-2/gunicorn.sock

Django environment:
 /opt/inventory-system-2/.env
```

---

## Security

- Run the application as `inventory2`, not root.
- Use the server's system SSH user for deployment.
- Use a GitHub Deploy Key for private repository access.
- Store secrets in Ansible Vault.
- Do not commit `.env`.
- Do not commit private SSH keys or `.pem` files.
- Use UFW to restrict incoming traffic.
- Keep Gunicorn behind Nginx.
- Use `DEBUG=False` in production.

---

## Idempotency

The roles are designed to be idempotent.

Running the deployment repeatedly should not unnecessarily recreate
users, directories, databases, virtual environments, or services.

```bash
ansible-playbook playbooks/deploy_inventory.yml --ask-vault-pass
```

---

## Troubleshooting

### Ansible connection failure

```bash
ansible all -m ping
```

Check the private `inventory` for the correct:

- Server address
- SSH user
- Private key
- Sudo access

### GitHub authentication failure

```bash
ssh -T git@github.com
```

Check that the generated public key is configured as a GitHub Deploy Key.

### PostgreSQL dependency error

If `pg_config.h` is missing:

```bash
sudo apt install -y libpq-dev
```

### Django permission error

Check:

```bash
ls -ld /opt/inventory-system-2/logs
```

The application directories should be owned by:

```text
inventory2:inventory2
```

### Gunicorn failure

```bash
sudo systemctl status gunicorn
sudo journalctl -u gunicorn -n 100 --no-pager
```

### Nginx 502

```bash
sudo systemctl status gunicorn
ls -l /opt/inventory-system-2/gunicorn.sock
sudo nginx -t
sudo journalctl -u nginx -n 100 --no-pager
```

---

## Deployment Result

The automation was successfully validated on a local Ubuntu VM and an
AWS EC2 Ubuntu server.

Final AWS Ansible result:

```text
ok=66
changed=32
unreachable=0
failed=0
skipped=0
```

Application validation included:

- Frontend loading
- Authentication
- Dashboard
- Product management
- Stock management
- Stock Report
- API communication
- JWT authentication

---

## Project Links

**Application:**  
https://github.com/Saleemalik/inventory_system

**Ansible:**  
https://github.com/Saleemalik/inventory-ansible

---

## Author

**Saleem Malik**

Cloud & DevOps Capstone Project