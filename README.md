# Deploy Django on VPS (Hostinger)

This guide provides step-by-step instructions for deploying a **Django project** on a VPS using **Gunicorn**, **Nginx**, and **PostgreSQL**.  
It also includes optional setup for **auto deployment with GitHub Actions**.

---

## Prerequisites

- A VPS (Hostinger in this example)
- A Django project on GitHub
- Basic knowledge of Linux commands

---

## 1. Generate SSH Key on Local Machine

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

- Leave the passphrase empty.
- Public key: `~/.ssh/id_ed25519.pub` → add to your VPS.
- Private key: `~/.ssh/id_ed25519` → store in GitHub Actions repository secret `VPS_SSH_KEY`.

---

## 2. Connect to VPS

```bash
ssh root@123.123.123.123
```

> ⚠️ For security, it’s recommended to create a non-root user for deployment.

---

## 3. Install Astral UV

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

After installation, run `exit` and reconnect to your VPS.

---

## 4. Setup Project Directory

```bash
sudo mkdir -p /var/www
sudo chown -R $USER:$USER /var/www
cd /var/www
```

---

## 5. Clone Project & Install Dependencies

```bash
git clone https://github.com/prave-com/deploy-django
cd deploy-django
uv sync
source .venv/bin/activate
```

---

## 6. Configure Environment

```bash
cp .env.example .env
sudo nano .env
```

Set the following:

- `DEBUG=False`
- `SECRET_KEY` → generate with:
  ```bash
  python -c 'from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())'
  ```
- `ALLOWED_HOSTS` → your domain or VPS IP

---

## 7. Create Systemd Service

Create service file:

```bash
sudo nano /etc/systemd/system/deploy-django.service
```

Add:

```ini
[Unit]
Description=Gunicorn instance for Deploy Django
After=network.target

[Service]
User=root
Group=www-data
WorkingDirectory=/var/www/deploy-django
Environment="PATH=/var/www/deploy-django/.venv/bin"
ExecStart=/var/www/deploy-django/.venv/bin/gunicorn --workers 3 --bind unix:/var/www/deploy-django/deploy-django.sock config.wsgi:application

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> ⚠️ In this configuration, there is `config.wsgi:application` in `ExecStart`. Change the `config` with your directory name, where the `wsgi.py` located.

---

## 8. Configure Nginx

Create config file, change `example.com` with your domain or VPS IP address:

```bash
sudo nano /etc/nginx/sites-available/example.com
```

Example configuration:

```nginx
server {
    listen 80;
    server_name example.com;

    location /static/ {
        alias /var/www/deploy-django/staticfiles/;
    }

    location /media/ {
        alias /var/www/deploy-django/media/;
    }

    location / {
        proxy_pass http://unix:/var/www/deploy-django/deploy-django.sock;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
```

> To enable HTTPS with Let’s Encrypt, if you don't have domain, skip this:

```bash
sudo certbot --nginx -d example.com
```

Reload Nginx:

```bash
sudo systemctl reload nginx
```

---

## 9. Configure Static & Media Files

In `settings.py`:

```python
STATIC_URL = "static/"
STATIC_ROOT = BASE_DIR / "staticfiles"

MEDIA_URL = "/media/"
MEDIA_ROOT = BASE_DIR / "media"
```

---

## 10. Install & Configure PostgreSQL

### Install PostgreSQL 16

```bash
sudo apt install wget ca-certificates -y
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O /etc/apt/trusted.gpg.d/postgresql.asc https://www.postgresql.org/media/keys/ACCC4CF8.asc
sudo apt update
sudo apt install postgresql-16 postgresql-client-16 -y
```

### Create Database & User

```bash
sudo -u postgres psql
```

```sql
CREATE DATABASE deploy_django;
CREATE USER deploy_django_user WITH PASSWORD 'your_password';
ALTER ROLE deploy_django_user SET client_encoding TO 'utf8';
ALTER ROLE deploy_django_user SET default_transaction_isolation TO 'read committed';
ALTER ROLE deploy_django_user SET timezone TO 'UTC';
GRANT ALL PRIVILEGES ON DATABASE deploy_django TO deploy_django_user;
```

```sql
\c deploy_django
```

```sql
GRANT ALL ON SCHEMA public TO deploy_django_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO deploy_django_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO deploy_django_user;
```

### Update `.env`

```
DB_HOST=localhost
DB_PORT=5432
DB_NAME=deploy_django
DB_USER=deploy_django_user
DB_PASSWORD=your_password
```

---

## 11. Run Migrations & Collect Static Files

```bash
python manage.py migrate
python manage.py collectstatic
python manage.py createsuperuser  # optional
```

---

## 12. Start Django Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable deploy-django
sudo systemctl start deploy-django
```

---

## 13. Auto Deploy with GitHub Actions (Optional)

1. Add GitHub repository secrets:

   - `VPS_HOST` → VPS IP
   - `VPS_USER` → SSH user
   - `VPS_SSH_KEY` → private key in your local computer (`~/.ssh/id_ed25519`)

2. Generate deploy key:

```bash
ssh-keygen -t rsa -b 4096 -C "deploy@deploy-django" -f ~/.ssh/github_deploy_django_rsa
cat ~/.ssh/github_deploy_django_rsa.pub
chmod 600 ~/.ssh/github_deploy_django_rsa
```

3. Configure SSH:

```bash
sudo nano ~/.ssh/config
```

```
Host deploy-django
  HostName github.com
  User git
  IdentityFile ~/.ssh/github_deploy_django_rsa
  IdentitiesOnly yes
```

4. Update remote repository:

```bash
git remote set-url origin git@deploy-django:prave-com/deploy-django.git
```

5. Add GitHub to known hosts:

```bash
ssh-keyscan github.com >> ~/.ssh/known_hosts
```

6. Test connection:

```bash
ssh -T git@deploy-django
```

---

✅ Your Django project should now be running in production on your VPS.
