
Short, practical checklist and examples for connecting a domain to a server with SSL, PostgreSQL, Nginx, basic security, backups, and deployment. Replace placeholders (`example.com`, `11.22.33.44`, `db_user`, `db_name`, `db_pass`) when applying to your environment.

## Table of Contents
- [devops-notes](#devops-notes)
  - [DevOps Notes](#devops-notes-1)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Quick Setup](#quick-setup)
  - [Local SSH Key \& Connect (mac)](#local-ssh-key--connect-mac)
  - [Server: Update, Create Directories \& Install Packages](#server-update-create-directories--install-packages)
  - [Pull and Deploy Repository](#pull-and-deploy-repository)
    - [Frontend project (Next.js) — example](#frontend-project-nextjs--example)
    - [Backend project (NestJS) — example](#backend-project-nestjs--example)
  - [DNS \& Networking](#dns--networking)
  - [Server Prerequisites](#server-prerequisites)
  - [PostgreSQL Setup](#postgresql-setup)
  - [Nginx + Reverse Proxy + HTTP-first flow and Certbot](#nginx--reverse-proxy--http-first-flow-and-certbot)
  - [Obtain SSL (Let's Encrypt)](#obtain-ssl-lets-encrypt)
  - [Systemd Service Example](#systemd-service-example)
  - [Backups](#backups)
  - [Security Hardening](#security-hardening)
  - [Deploying Your App](#deploying-your-app)
  - [Database Migrations \& Rollback](#database-migrations--rollback)
  - [Monitoring \& Logging](#monitoring--logging)
  - [Connect Domain](#connect-domain)
    - [Using a CDN (example: Cloudflare)](#using-a-cdn-example-cloudflare)
    - [Setting up DNS with BIND9 (self-hosted DNS)](#setting-up-dns-with-bind9-self-hosted-dns)
  - [PostgreSQL Backup \& Restore](#postgresql-backup--restore)
    - [1) Create backup from local machine](#1-create-backup-from-local-machine)
    - [2) Transfer backup to server using SCP](#2-transfer-backup-to-server-using-scp)
    - [3) Restore backup on the server](#3-restore-backup-on-the-server)
    - [4) Backup table-by-table (optional)](#4-backup-table-by-table-optional)
    - [5) Automated backup scheduling](#5-automated-backup-scheduling)
  - [Troubleshooting \& Useful Commands](#troubleshooting--useful-commands)
  - [References](#references)

## Introduction
Concise, repeatable steps to connect `example.com` -> `11.22.33.44` with TLS, run a web app behind Nginx, and use PostgreSQL. Focus on reproducibility and safety.

## Quick Setup
- **Domain:** Point an A record for `example.com` to `11.22.33.44`.
- **User:** Create a non-root sudo user on the server (`deploy`).
- **SSH:** Use key-based auth and disable password auth.

## Local SSH Key & Connect (mac)
Step-by-step to create an SSH key on macOS and add it to the server.

```bash
# 1. Create an Ed25519 key (change email and filename as needed)
ssh-keygen -t ed25519 -C "your_email@example.com" -f ~/.ssh/id_ed25519_example -N ""

# 2. Show the public key (copy this into server's ~/.ssh/authorized_keys)
cat ~/.ssh/id_ed25519_example.pub

# 3a. Preferred: Use ssh-copy-id to install the pubkey on the server
# (installs into deploy@11.22.33.44 ~/.ssh/authorized_keys)
ssh-copy-id -i ~/.ssh/id_ed25519_example.pub deploy@11.22.33.44

# 3b. Alternative (manual): paste the output of the cat command into the server
# via an interactive SSH session or using a one-liner:
cat ~/.ssh/id_ed25519_example.pub | ssh deploy@11.22.33.44 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys'

# 4. Connect using the new key
ssh -i ~/.ssh/id_ed25519_example deploy@11.22.33.44
```

## Server: Update, Create Directories & Install Packages
Commands to bootstrap an Ubuntu server. Run as `deploy` with `sudo` or as `root` when appropriate.

```bash
# 1. Update system packages
sudo apt update && sudo apt upgrade -y

# 2. Create a WWW directory for hosting (change owner to deploy)
sudo mkdir -p /var/www
sudo chown -R $(whoami):$(whoami) /var/www

# 3. Install Git
sudo apt install -y git

# 4. Install Node.js (Node 18 LTS example)
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs build-essential

# 5. Install Nginx
sudo apt install -y nginx

# 6. Install PM2 (global process manager)
sudo npm install -g pm2

# 7. Optional: Install PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# 8. Optional: Install Redis
sudo apt install -y redis-server

# 9. Post-install: ensure Nginx and services are enabled
sudo systemctl enable --now nginx
```

Notes:
- If you use a different distro, replace `apt` commands accordingly.
- Consider locking down `postgresql` and `redis` bind addresses in their configs (e.g., `listen_addresses = 'localhost'`).

## Pull and Deploy Repository
Generic flow: clone into `/var/www` (or `/home/deploy`) and run using `pm2`. Run these as the `deploy` user.

### Frontend project (Next.js) — example

```bash
# 1. Clone (replace repo URL)
cd /var/www
git clone git@github.com:your/repo-frontend.git frontend
cd frontend

# 2. Install dependencies
npm ci

# 3. Build for production
npm run build

# 4. Start with PM2 (use the package.json start script or next start)
# If `package.json` has a `start` script that runs `next start`:
pm2 start npm --name frontend -- start

# 5. Save PM2 process list and enable startup on reboot
pm2 save
pm2 startup systemd -u $(whoami) --hp /home/$(whoami)

# 6. Update / redeploy (pull new commits, reinstall if needed, rebuild, restart)
git pull origin main
npm ci
npm run build
pm2 restart frontend
```

### Backend project (NestJS) — example

```bash
# 1. Clone (replace repo URL)
cd /var/www
git clone git@github.com:your/repo-backend.git backend
cd backend

# 2. Install dependencies
npm ci

# 3. Setup .env (example - create / edit .env file)
cat > .env <<EOF
PORT=3000
DATABASE_URL=postgresql://db_user:db_pass@localhost:5432/db_name
NODE_ENV=production
EOF

# 4. Build the project
npm run build

# 5. Run with PM2 (start the built JS entrypoint, often dist/main.js)
pm2 start dist/main.js --name backend --update-env

# 6. Save PM2 list and enable on boot
pm2 save
pm2 startup systemd -u $(whoami) --hp /home/$(whoami)

# 7. Update / redeploy (pull, install, build, restart)
git pull origin main
npm ci
npm run build
pm2 restart backend
```

Notes:
- Adjust paths and `pm2` start command if your app uses a different entrypoint or expects `npm start`.
- For environment management, prefer `pm2 ecosystem` files or `systemd` unit files for production-critical services.


## DNS & Networking
- **A record:** `example.com A 11.22.33.44`.
- **Propagation check:** `dig +short example.com` → should return `11.22.33.44`.
- **Ports:** Open TCP 22 (SSH), 80 (HTTP), 443 (HTTPS) on your cloud provider and firewall.

## Server Prerequisites
- **OS:** Assume Ubuntu LTS (adjust for other distros).
- **Packages:** `sudo apt update && sudo apt install -y nginx postgresql certbot python3-certbot-nginx git ufw`.

## PostgreSQL Setup
Create a secure DB user and database (example):

```bash
sudo -u postgres psql -c "CREATE USER db_user WITH PASSWORD 'db_pass';"
sudo -u postgres psql -c "CREATE DATABASE db_name OWNER db_user;"
```

- **Connection string:** `postgresql://db_user:db_pass@localhost:5432/db_name`
- **pg_hba.conf:** Use `scram-sha-256` or `md5` and restrict host access.

## Nginx + Reverse Proxy + HTTP-first flow and Certbot
Nginx acts as a reverse proxy for the frontend and backend. Below are step-by-step instructions to create a new site configuration, enable it, test, and reload Nginx.

Follow this flow when you want to start by serving HTTP (quick verification) and then add TLS with Certbot.

1) Create a simple HTTP-only site for initial testing

Create `/etc/nginx/sites-available/myapp` with a minimal HTTP server block that proxies to your frontend and backend. This lets you verify routing before obtaining certificates.
```bash
sudo nano /etc/nginx/sites-available/myapp
```

```nginx
server {
  listen 80;
  server_name example.com;

  # Frontend (Next.js)
  location / {
    proxy_pass http://localhost:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
  }

  # Backend (NestJS)
  location /api {
    proxy_pass http://localhost:8000/api;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;

    # Optional: handle preflight requests (CORS)
    if ($request_method = OPTIONS) {
      add_header 'Access-Control-Allow-Origin' '*';
      add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, DELETE';
      add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type';
      add_header 'Access-Control-Max-Age' 86400;
      add_header 'Content-Length' 0;
      add_header 'Content-Type' 'text/plain';
      return 204;
    }
  }
}
```

Then enable and test:

```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/example.com
sudo nginx -t
sudo systemctl reload nginx
```

Visit `http://example.com` to verify the frontend, and test API routes at `http://example.com/api/...`.

2) Install Certbot and obtain certificates

Install Certbot and the Nginx plugin, then request certificates. Certbot will (by default) try to update your Nginx config to use the certs — you can also use the webroot method.

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y

# Obtain cert (nginx plugin):
sudo certbot --nginx -d example.com

# Test auto-renewal:
sudo certbot renew --dry-run
```

3) Final HTTPS + redirect config (exact example)

After obtaining certificates, update `/etc/nginx/sites-available/myapp` to the HTTPS configuration below (or let Certbot modify it). This config redirects HTTP to HTTPS and configures the proxy rules and OPTIONS handling.

```nginx
# Redirect all HTTP traffic to HTTPS
server {
  listen 80;
  server_name example.com;

  # Optional: redirect everything to HTTPS
  return 301 https://$host$request_uri;
}

# Handle HTTPS traffic
server {
  listen 443 ssl;
  server_name example.com;

  ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

  # Frontend (Next.js)
  location / {
    proxy_pass http://localhost:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
  }

  # Backend (NestJS)
  location /api {
    proxy_pass http://localhost:8000/api;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;

    # Handle preflight requests (CORS)
    if ($request_method = OPTIONS) {
      add_header 'Access-Control-Allow-Origin' 'https://example.com';
      add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS, PUT, DELETE';
      add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type';
      add_header 'Access-Control-Max-Age' 86400;
      add_header 'Content-Length' 0;
      add_header 'Content-Type' 'text/plain';
      return 204;
    }
  }
}
```

Reload Nginx after editing the file:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Notes:
- If Certbot modified your config automatically, inspect `/etc/nginx/sites-available/myapp` to ensure the proxy settings remained correct (Certbot sometimes adds its own server blocks).
- For stricter security, consider adding HSTS headers and stronger TLS settings (see earlier SSL snippet).


## Obtain SSL (Let's Encrypt)
Use Certbot with the Nginx plugin to get and auto-renew certificates:

```bash
sudo certbot --nginx -d example.com --email admin@example.com --agree-tos --no-eff-email
```

Verify renewal with: `sudo certbot renew --dry-run`.

## Systemd Service Example
Example `systemd` unit for a Node/Express app that listens on localhost:3000.

File `/etc/systemd/system/myapp.service`:

```ini
[Unit]
Description=My App
After=network.target

[Service]
User=deploy
WorkingDirectory=/home/deploy/myapp
ExecStart=/usr/bin/node /home/deploy/myapp/index.js
Restart=on-failure
Environment=NODE_ENV=production
Environment=DATABASE_URL="postgresql://db_user:db_pass@localhost:5432/db_name"

[Install]
WantedBy=multi-user.target
```

Enable and start:
`sudo systemctl daemon-reload && sudo systemctl enable --now myapp`

## Backups
- **Logical backup (daily):**

```bash
PGPASSWORD=db_pass pg_dump -U db_user -h localhost db_name | gzip > /var/backups/db_name-$(date +%F).sql.gz
```

- Automate with a cron job and rotate older backups (or push to remote storage with `rsync`/S3).

## Security Hardening
- **UFW example:**

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'   # opens 80 and 443
sudo ufw enable
```

- **SSH:** Disable password auth in `/etc/ssh/sshd_config`: `PasswordAuthentication no` and restart `sshd`.
- **Fail2ban:** Install and enable to mitigate brute force attacks.
- **Keep packages updated:** `sudo apt upgrade` regularly or use unattended-upgrades.

## Deploying Your App
- **Clone:** `git clone git@github.com:your/repo.git /home/deploy/myapp`
- **Env:** store secrets in a `.env` file or use system environment (never commit secrets).
- **Start:** Use `systemd` unit above or process manager like `pm2`/`supervisord`.

## Database Migrations & Rollback
- Prefer a migrations tool (Flyway, Alembic, Django migrations, Knex, etc.).
- Always run migrations on a staging environment first and keep backups before applying to production.

## Monitoring & Logging
- **Logs:** Use `journalctl -u myapp` and rotate access/error logs in Nginx.
- **Monitoring:** Add simple healthcheck endpoints and integrate with Prometheus/Grafana or a hosted monitoring service.

## Connect Domain

Two common approaches: use a CDN (Cloudflare, AWS CloudFront) or point DNS directly to your server IP. Below are both methods.

### Using a CDN (example: Cloudflare)

1) Register your domain with a registrar or use an existing one.

2) In your domain registrar, update nameservers to point to Cloudflare's nameservers (Cloudflare will provide them during signup).

3) In Cloudflare dashboard, add an A record pointing to your server:
   - **Type:** A
   - **Name:** `example.com` (or `@` for root)
   - **IPv4 address:** `11.22.33.44`
   - **Proxied:** Yes (orange cloud) or No (gray cloud, direct to server)

4) Enable SSL in Cloudflare dashboard (Crypto tab) — set to "Full (strict)" or "Full" depending on your cert setup.

5) Test DNS propagation:

```bash
dig example.com
# or
nslookup example.com
```

Benefits of CDN: DDoS protection, edge caching, WAF rules.

### Setting up DNS with BIND9 (self-hosted DNS)

If you want to self-host DNS on your server, install and configure BIND9:

1) Install BIND9:

```bash
sudo apt update
sudo apt install -y bind9 bind9-utils bind9-doc
```

2) Create a zone file. Edit `/etc/bind/zones/db.example.com`:

```bash
sudo mkdir -p /etc/bind/zones
sudo nano /etc/bind/zones/db.example.com
```

Add the following content (replace `11.22.33.44` with your server IP and update `ns1.example.com` as needed):

```
$TTL    604800
@       IN      SOA     ns1.example.com. admin.example.com. (
                        2024112701      ; Serial
                        604800          ; Refresh
                        86400           ; Retry
                        2419200         ; Expire
                        604800)         ; Minimum TTL

@       IN      NS      ns1.example.com.
@       IN      NS      ns2.example.com.

@       IN      A       11.22.33.44
ns1     IN      A       11.22.33.44
ns2     IN      A       11.22.33.44
www     IN      A       11.22.33.44
```

3) Configure BIND9 named.conf. Edit `/etc/bind/named.conf.local` and add:

```bash
sudo nano /etc/bind/named.conf.local
```

Add:

```
zone "example.com" {
    type master;
    file "/etc/bind/zones/db.example.com";
    allow-transfer { any; };
};
```

4) Check BIND9 config syntax:

```bash
sudo named-checkconf
sudo named-checkzone example.com /etc/bind/zones/db.example.com
```

5) Enable and start BIND9:

```bash
sudo systemctl enable --now bind9
sudo systemctl status bind9
```

6) Test local DNS resolution:

```bash
nslookup example.com 127.0.0.1
```

7) Update your domain registrar to use your nameservers (e.g., `ns1.example.com`, `ns2.example.com`).

8) Test public DNS propagation:

```bash
dig @8.8.8.8 example.com
```

## PostgreSQL Backup & Restore

Step-by-step guide to backup, transfer, and restore PostgreSQL databases.

### 1) Create backup from local machine

If PostgreSQL is running on your local machine, back up the database(s):

```bash
# Backup a single database (prompts for password)
pg_dump -U db_user -h localhost db_name > ~/backups/db_name-$(date +%F).sql

# Backup a single database without prompt (if you trust .pgpass or environment)
PGPASSWORD=db_pass pg_dump -U db_user -h localhost db_name > ~/backups/db_name-$(date +%F).sql

# Backup entire cluster (all databases)
pg_dumpall -U postgres > ~/backups/all-databases-$(date +%F).sql

# Backup with compression (smaller file)
PGPASSWORD=db_pass pg_dump -U db_user -h localhost db_name | gzip > ~/backups/db_name-$(date +%F).sql.gz
```

Verify the backup was created:

```bash
ls -lh ~/backups/
```

### 2) Transfer backup to server using SCP

From your local machine (mac), copy the backup file to the server:

```bash
# Single backup file
scp -i ~/.ssh/id_ed25519_example ~/backups/db_name-2024-11-27.sql deploy@11.22.33.44:/tmp/

# Multiple backup files
scp -i ~/.ssh/id_ed25519_example ~/backups/*.sql deploy@11.22.33.44:/tmp/

# Compressed backup
scp -i ~/.ssh/id_ed25519_example ~/backups/db_name-2024-11-27.sql.gz deploy@11.22.33.44:/tmp/
```

Verify on the server:

```bash
ssh -i ~/.ssh/id_ed25519_example deploy@11.22.33.44
ls -lh /tmp/db_name-*.sql*
```

### 3) Restore backup on the server

Connect to the server and restore the database:

```bash
# SSH into server
ssh -i ~/.ssh/id_ed25519_example deploy@11.22.33.44

# Restore a single database (drops and recreates)
sudo -u postgres psql < /tmp/db_name-2024-11-27.sql

# Or if you prefer to restore with a specific user/password:
PGPASSWORD=db_pass psql -U db_user -h localhost db_name < /tmp/db_name-2024-11-27.sql

# Restore from compressed backup
gunzip -c /tmp/db_name-2024-11-27.sql.gz | sudo -u postgres psql

# Restore all databases (full cluster restore)
sudo -u postgres psql < /tmp/all-databases-2024-11-27.sql
```

Verify the restore succeeded:

```bash
# List databases
sudo -u postgres psql -l

# Connect to the restored database and check tables
sudo -u postgres psql -d db_name -c '\dt'
```

### 4) Backup table-by-table (optional)

For large databases, you can back up individual tables:

```bash
# Backup a single table
PGPASSWORD=db_pass pg_dump -U db_user -h localhost -t table_name db_name > ~/backups/table_name-$(date +%F).sql

# List all tables in a database (to decide which to back up)
PGPASSWORD=db_pass psql -U db_user -h localhost db_name -c '\dt'
```

Then restore that table:

```bash
# On server
PGPASSWORD=db_pass psql -U db_user -h localhost db_name < /tmp/table_name-2024-11-27.sql
```

### 5) Automated backup scheduling

On the server, create a cron job to back up PostgreSQL daily:

```bash
# Edit crontab
crontab -e

# Add this line to back up at 2 AM daily
0 2 * * * PGPASSWORD=db_pass pg_dump -U db_user -h localhost db_name | gzip > /var/backups/db_name-$(date +\%F).sql.gz

# Clean old backups (keep last 7 days)
0 3 * * * find /var/backups/db_name-*.sql.gz -mtime +7 -delete
```

Verify cron is running:

```bash
crontab -l
```

## Troubleshooting & Useful Commands
- **Check Nginx config:** `sudo nginx -t` and `sudo systemctl restart nginx`.
- **Check service logs:** `sudo journalctl -u myapp -f`.
- **Check disk usage:** `df -h` and `du -sh /var/log/*`.
- **Postgres connection test:** `psql postgresql://db_user:db_pass@localhost:5432/db_name -c '\l'`

## References
- Certbot: https://certbot.eff.org/
- PostgreSQL docs: https://www.postgresql.org/docs/
- Nginx docs: https://nginx.org/en/docs/
