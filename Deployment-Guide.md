# CMACTS Deployment Guide
## Laravel 9 API + Nuxt.js 2 Frontend
### Server: Ubuntu 24.04 LTS (Contabo VPS)

---

## 📋 Cover Topics & Table of Contents
1. **[Server Preparation](#step-2-update-server)** (System updates and basic package installations)
2. **[MySQL Installation & Management](#step-5-install-mysql-server)** (Installing MySQL Server, creating databases, and users)
3. **[PHP 8.3 Setup](#step-3-install-php-83--php-fpm)** (Installing PHP, PHP-FPM, and critical extensions)
4. **[Composer](#step-4-install-composer)** (Installing Composer package manager)
5. **[Redis](#step-6-install-redis)** (In-memory database server setup)
6. **[NodeJS 22](#step-7-install-nodejs-22-lts)** (Installing Node and npm)
7. **[PM2 Installation](#step-8-install-pm2)** (Process manager configuration)
8. **[Laravel Deployment](#step-10-deploy-laravel-backend)** (Setting up .env, permissions, migrations, optimizations, and storage links)
9. **[Nuxt Deployment](#step-12-deploy-nuxtjs-2-frontend)** (Frontend dependency assembly, environments, and PM2 deployment)
10. **[Nginx Configuration](#step-13-configure-api-nginx)** (Setting up reverse proxy rules for API and web clients)
11. **[SSL/Certbot](#step-15-install-ssl)** (Acquiring Let's Encrypt certificates)
12. **[Cloudflare Troubleshooting](#cloudflare-flexible-ssl-warning)** (Preventing flexible redirect loops)
13. **[Queue Worker Setup](#step-11-configure-queue-worker)** (Configuring Supervisor processes)
14. **[Verification Checklist](#verification)** (Command status and URL validation tests)
15. **[Reset/Cleanup Procedure](#complete-vps-reset-uninstall--purge-package-guide)** (Complete system package purge and file erasure guidelines)
16. **[Common Troubleshooting Commands](#troubleshooting--verification)** (Monitoring logs and service restarts)

---

## Environment Information

| Component | URL |
|------------|------------|
| API | https://test-cmacts-api.orangebd.com |
| Frontend | https://test-cmacts.orangebd.com |

### Application Structure

```text
/var/www/cmacts
├── backend     (Laravel 9 API)
└── frontend    (Nuxt.js 2)
```

## Step 1: Verify Server Environment

```bash
php -v
node -v
npm -v
pm2 -v
nginx -v
supervisord -v
```

Expected:

```text
PHP 8.3.x
Node 22.x
NPM 10+
PM2 6+
Nginx 1.24+
Supervisor 4+
```

## Step 2: Update Server

```bash
apt update
apt upgrade -y
```

Install required packages:

```bash
apt install -y \
git \
curl \
wget \
zip \
unzip \
nginx \
supervisor \
software-properties-common
```

Enable, start, and check status of Nginx:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
```

## Step 3: Install PHP 8.3 + PHP-FPM

```bash
apt install -y \
php8.3 \
php8.3-cli \
php8.3-fpm \
php8.3-common \
php8.3-mysql \
php8.3-curl \
php8.3-mbstring \
php8.3-xml \
php8.3-zip \
php8.3-bcmath \
php8.3-gd \
php8.3-intl
```

```bash
systemctl enable php8.3-fpm
systemctl start php8.3-fpm
```

Verify socket:

```bash
find /run/php -name "*.sock"
```

Expected:

```text
/run/php/php8.3-fpm.sock
```

## Step 4: Install Composer

```bash
apt install composer -y
composer -V
```

## Step 5: Install MySQL Server

```bash
sudo apt update
sudo apt install mysql-server -y
```

After installation:

```bash
sudo systemctl enable mysql
sudo systemctl start mysql
sudo systemctl status mysql
```

Login:

```bash
sudo mysql
```

Then create the database and user:

```sql
CREATE DATABASE cmacts
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;

CREATE USER 'cmacts_user'@'localhost'
IDENTIFIED BY 'StrongPassword123!';

GRANT ALL PRIVILEGES ON cmacts.* TO 'cmacts_user'@'localhost';

FLUSH PRIVILEGES;
EXIT;
```

Verify:

```bash
mysql -u cmacts_user -p
```

Then:

```sql
SHOW DATABASES;
USE cmacts;
```

## Step 6: Install Redis

```bash
apt install redis-server -y

systemctl enable redis-server
systemctl start redis-server

redis-cli ping
```

Expected:

```text
PONG
```

## Step 7: Install NodeJS 22 LTS

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | bash -

apt install -y nodejs

node -v
npm -v
```

## Step 8: Install PM2

```bash
npm install -g pm2
pm2 -v
```

## Step 9: Create Application Directory

```bash
mkdir -p /var/www/cmacts
cd /var/www/cmacts
```

## Step 10: Deploy Laravel Backend

```bash
cd /var/www/cmacts

git clone YOUR_BACKEND_REPOSITORY backend

cd backend

composer install --no-dev --optimize-autoloader

cp .env.example .env
```

Update .env:

```env
APP_NAME=CMACTS
APP_ENV=production
APP_DEBUG=false
APP_DOMAIN=https://test-cmacts-api.orangebd.com
APP_URL="${APP_DOMAIN}"
FRONTEND_URL=https://test-cmacts.orangebd.com
SANCTUM_STATEFUL_DOMAINS="${APP_DOMAIN}"

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=
DB_USERNAME=
DB_PASSWORD=

# Use redis if possible
BROADCAST_DRIVER=log
CACHE_DRIVER=file
QUEUE_CONNECTION=database
SESSION_DRIVER=file
SESSION_LIFETIME=120

TELESCOPE_ENABLED=false

# Philippine SMS Gateway (Smart)
SMS_API_ENDPOINT=""
SMS_API_ID=""
SMS_API_KEY=""
SMS_COUNTRY_CODE=""
SMS_SENDER_ID=""
```

```bash
php artisan key:generate

php artisan optimize:clear

php artisan migrate

chown -R www-data:www-data /var/www/cmacts/backend

chmod -R 775 storage
chmod -R 775 bootstrap/cache

php artisan storage:link

```

## Step 11: Configure Queue Worker

Create:

```bash
nano /etc/supervisor/conf.d/cmacts-worker.conf
```

```ini
[program:cmacts-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/cmacts/backend/artisan queue:work --sleep=3 --tries=3
autostart=true
autorestart=true
numprocs=2
user=www-data
redirect_stderr=true
stdout_logfile=/var/log/cmacts-worker.log
```

```bash
supervisorctl reread
supervisorctl update
supervisorctl start cmacts-worker:*
sudo supervisorctl restart all
```

## Step 12: Deploy Nuxt.js 2 Frontend

```bash
cd /var/www/cmacts

git clone YOUR_FRONTEND_REPOSITORY frontend

cd frontend

npm install
```

Create .env:

```env
BASE_URL=https://test-cmacts.orangebd.com
API_URL=https://test-cmacts-api.orangebd.com/api/v1
```

<!-- Add to nuxt.config.js:

```js
export default {
  server: {
    host: '0.0.0.0',
    port: 3000
  }
} -->
```

Build and start:

```bash
npm run build

pm2 start npm --name cmacts-frontend -- start

pm2 save
pm2 startup
```

## Step 13: Configure API Nginx

```nginx
server {
    listen 80;
    server_name test-cmacts-api.orangebd.com;

    root /var/www/cmacts/backend/public;
    index index.php index.html;

    client_max_body_size 100M;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }

    location ~ /\. {
        deny all;
    }
}
```

## Step 14: Configure Frontend Nginx

```nginx
server {
    listen 80;
    server_name test-cmacts.orangebd.com;

    location / {
        proxy_pass http://localhost:3000;

        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 300;
        proxy_connect_timeout 300;
    }
}
```

Enable:

```bash
ln -s /etc/nginx/sites-available/cmacts-api /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/cmacts-web /etc/nginx/sites-enabled/

rm -f /etc/nginx/sites-enabled/default

nginx -t
systemctl reload nginx
```

## Step 15: Install SSL

```bash
apt install certbot python3-certbot-nginx -y

certbot --nginx -d test-cmacts-api.orangebd.com

certbot --nginx -d test-cmacts.orangebd.com
```

## Cloudflare Flexible SSL Warning

If Cloudflare SSL mode is Flexible, Certbot's HTTP → HTTPS redirect may create an infinite redirect loop.

Recommended Cloudflare mode:

```text
Full (Strict)
```

Temporary workaround: remove the HTTP→HTTPS redirect generated by Certbot until Cloudflare is updated.

## Verification

```bash
php -v
node -v
npm -v

pm2 list
supervisorctl status

nginx -t

systemctl status nginx
systemctl status php8.3-fpm

curl -Ik https://localhost -H "Host: test-cmacts.orangebd.com"
curl -Ik https://localhost -H "Host: test-cmacts-api.orangebd.com"
```

## Complete VPS Reset (Uninstall & Purge Package Guide)

If you need to test the entire deployment process from scratch, use these commands to completely uninstall and wipe Nginx, PHP, Node.js, PM2, Supervisor, and MySQL from the VPS.

### 1. Stop Services and Delete Project Files
```bash
# Stop PM2 processes and daemon
pm2 kill || true

# Stop Supervisor workers
supervisorctl stop all || true

# Delete all application directories
rm -rf /var/www/cmacts
```

### 2. Completely Uninstall Nginx
```bash
# Stop and disable service
sudo systemctl stop nginx || true
sudo systemctl disable nginx || true

# Purge packages
sudo apt-get purge nginx nginx-common nginx-core -y

# Delete configurations and directories
sudo rm -rf /etc/nginx /var/log/nginx /var/www/html
```

### 3. Completely Uninstall PHP & Composer
```bash
# Stop and disable PHP-FPM
sudo systemctl stop php8.3-fpm || true
sudo systemctl disable php8.3-fpm || true

# Purge PHP packages
sudo apt-get purge php8.3* php-common -y

# Delete Composer
sudo rm -f /usr/local/bin/composer /usr/bin/composer

# Delete configuration files
sudo rm -rf /etc/php /var/log/php*
```

### 4. Completely Uninstall Node.js, npm, & PM2
```bash
# Uninstall PM2 globally
npm uninstall -g pm2 || true

# Purge Node.js and npm
sudo apt-get purge nodejs npm -y

# Remove configuration folders
rm -rf ~/.pm2 ~/.npm ~/.node-gyp
sudo rm -rf /usr/lib/node_modules /usr/local/lib/node_modules
sudo rm -rf /usr/local/bin/node /usr/local/bin/npm /usr/local/bin/pm2
```

### 5. Completely Uninstall Supervisor
```bash
# Stop and disable Supervisor
sudo systemctl stop supervisor || true
sudo systemctl disable supervisor || true

# Purge package
sudo apt-get purge supervisor -y

# Remove configs and worker log file
sudo rm -f /etc/supervisor/conf.d/cmacts-worker.conf
sudo rm -rf /etc/supervisor /var/log/supervisor /var/log/cmacts-worker.log
```

### 6. Completely Uninstall MySQL Server
```bash
# Stop and disable MySQL
sudo systemctl stop mysql || true
sudo systemctl disable mysql || true

# Purge packages
sudo apt-get purge mysql-server mysql-client mysql-common mysql-server-core-* mysql-client-core-* -y

# Delete configurations and data folders
sudo rm -rf /etc/mysql /var/lib/mysql /var/log/mysql
```

### 7. Clean Residual Packages and Cache
```bash
sudo apt-get autoremove -y
sudo apt-get clean
sudo apt-get autoclean
```

---

## Troubleshooting & Verification

To verify that all services have been wiped before starting again:

```bash
# Verify commands are not found (should return 'command not found')
nginx -v
php -v
node -v
mysql --version
supervisorctl status

# Check running services (should return inactive/not-found)
systemctl status nginx
systemctl status php8.3-fpm
systemctl status mysql
systemctl status supervisor

# Monitor logs if needed during installation
tail -f /var/log/nginx/error.log
tail -f /var/log/cmacts-worker.log
```
