# 🚀 How to Host Django Application using Gunicorn & Nginx in Production

This guide walks you through deploying a Django project using **Gunicorn** as the WSGI server and **Nginx** as a reverse proxy on a Linux server.

---

## 📦 Step 1 - Install Python and Nginx

```bash
sudo apt update
sudo apt install python3-pip python3-dev nginx
```

This installs Python, pip, and the Nginx server.

---

## 🧪 Step 2 - Create a Python Virtual Environment

```bash
sudo apt install python3-venv
python3 -m venv venv
source venv/bin/activate
```

---

## ⚙️ Step 3 - Install Django and Gunicorn

```bash
pip install django gunicorn
```

This installs Django and Gunicorn inside your virtual environment.

---

## 🔧 Step 4 - Django Settings Configuration

Update your `settings.py`:

```python
ALLOWED_HOSTS = ['your-server-ip-or-domain']
DEBUG = False

STATIC_URL = '/static/'
STATICFILES_DIRS = [os.path.join(BASE_DIR, 'static')]
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')

MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
```

Run migrations if necessary:

```bash
python manage.py makemigrations
python manage.py migrate
```

Test the project:

```bash
sudo ufw allow 8000
python manage.py runserver 0.0.0.0:8000
```

Visit `http://your_ip:8000` in a browser to verify.

---

## 🎯 Step 5 - Configure Gunicorn

Start Gunicorn manually to verify:

```bash
gunicorn --bind 0.0.0.0:8000 your_project_name.wsgi
```

To daemonize Gunicorn, create a socket and service:

### 🔌 Create Socket File

```bash
sudo vim /etc/systemd/system/gunicorn.socket
```

Paste:

```ini
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```

### ⚙️ Create Service File

```bash
sudo vim /etc/systemd/system/gunicorn.service
```

Paste:

```ini
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=deployuser
Group=www-data
WorkingDirectory=/home/equity_master_project
ExecStart=/home/equity_master_project/venv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          equity_master.wsgi:application
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Enable Gunicorn:

```bash
sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket
```

---

## 🌐 Step 6 - Configure Nginx as a Reverse Proxy

```bash
sudo vim /etc/nginx/sites-available/equity_master
```

```nginx
server {
    listen 80;
    server_name your_server_ip_or_domain;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static/ {
        alias /home/equity_master_project/staticfiles/;
        expires 30d;
        add_header Cache-Control "public, max-age=2592000";
    }

    location /media/ {
        alias /home/equity_master_project/media/;
        expires 7d;
        add_header Cache-Control "public, max-age=604800";
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
```

Enable the configuration:

```bash
sudo ln -s /etc/nginx/sites-available/equity_master /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl daemon-reexec
sudo systemctl enable gunicorn.service
sudo systemctl start gunicorn.service
```

---

## 📝 Optional: Favicon

If you have a favicon, place it in the static directory and use:

```html
<link rel="icon" href="{% static 'favicon.ico' %}">
```

Then run:

```bash
python manage.py collectstatic
```

---

## 🔐 Best Practices and Explanations

### ✅ Why use `alias` and full paths in Nginx?

Using `alias` ensures Nginx can serve static/media files **directly** from the filesystem without touching Django. This is important because:

- Django is designed to handle **dynamic content**.
- Nginx is much faster at serving **static content** (images, CSS, JS).
- Reduces the load on your Django app.

We specify the complete path like `/home/equity_master_project/staticfiles/` so that the static and media requests go **directly through Nginx** — not Django — improving performance.

### 🧠 What does `expires` and `Cache-Control` do?

These headers tell the browser to **cache static/media files** for a fixed time:

```nginx
expires 30d;
add_header Cache-Control "public, max-age=2592000";
```

- This reduces repeated HTTP requests and speeds up page loads.
- You can bust the cache by renaming/changing the file path if the file is updated before 30 days.

### 👮 Limit Permissions

Ensure static/media folders are owned by the deploy user and readable by Nginx:

```bash
sudo chown -R deployuser:www-data /home/equity_master_project/staticfiles/
sudo chown -R deployuser:www-data /home/equity_master_project/media/

sudo chmod -R 755 /home/equity_master_project/staticfiles
sudo chmod -R 755 /home/equity_master_project/media

sudo chown -R deployuser:www-data /run/gunicorn.sock
```

### 🚫 Don't Run as root

Create user:

```bash
sudo adduser --system --group deployuser
```

Instead of:

```ini
User=root
```

Use:

```ini
User=deployuser
Group=www-data
```

This minimizes the risk if Gunicorn gets compromised.

### 📊 Log Monitoring

Check Gunicorn logs:

```bash
journalctl -u gunicorn.service
```

Check Nginx logs:

```bash
tail -f /var/log/nginx/access.log /var/log/nginx/error.log
```

---


# 🔐 Setting Permissions for Static and Media Files in Django Deployment

When deploying a Django application using **Gunicorn** and **Nginx**, it's important to configure the correct **ownership** and **permissions** for your static and media files to ensure both security and functionality.

---

## 📁 Why Set Permissions?

- **Static files** (CSS, JS, images) are collected and served directly by Nginx.
- **Media files** (user uploads) are saved by Django and also served by Nginx.
- These files must be **readable by Nginx** (`www-data` group) and **writable (if needed)** by Django (running as `deployuser`).

---

## 👤 Dedicated Deployment User

Instead of running Gunicorn as `root` (which is insecure), we use a dedicated system user:

```bash
# Create a system-level deploy user
sudo adduser --system --group deployuser
```

- This command creates:
  - A user: `deployuser`
  - A group: `deployuser` (same name)
- Use `deployuser` in your Gunicorn service file:

```ini
User=deployuser
Group=www-data  # Allows Nginx to read files
```

---

## 📁 Change Ownership and Permissions

### Set Ownership

```bash
sudo chown -R deployuser:www-data /home/equity_master_project/staticfiles
sudo chown -R deployuser:www-data /home/equity_master_project/media
```

- `deployuser`: owns the files (can write if needed)
- `www-data`: Nginx group that needs to read these files

### Set Permissions

```bash
sudo chmod -R 755 /home/equity_master_project/staticfiles
sudo chmod -R 755 /home/equity_master_project/media
```

- `755` means:
  - Owner: read, write, execute
  - Group: read, execute
  - Others: read, execute

This allows Nginx to **serve** the files but **not write** them — good for security.

---

## ⚠️ Don't Use `www-data:www-data` as Owner

Avoid this:

```bash
# ❌ Not recommended
sudo chown -R www-data:www-data /home/equity_master_project/staticfiles
```

This gives **write access** to the web server (Nginx), which is a **security risk**. Only Django (Gunicorn) should write files if necessary.

---

## ✅ Summary

| Action                        | Command (Example)                                                                 |
|------------------------------|-----------------------------------------------------------------------------------|
| Create deployment user       | `sudo adduser --system --group deployuser`                                       |
| Set file ownership           | `sudo chown -R deployuser:www-data /home/equity_master_project/staticfiles`      |
| Set media file ownership     | `sudo chown -R deployuser:www-data /home/equity_master_project/media`            |
| Set directory permissions    | `sudo chmod -R 755 /home/equity_master_project/staticfiles`                      |
|                              | `sudo chmod -R 755 /home/equity_master_project/media`                            |
| Gunicorn systemd User/Group  | `User=deployuser` <br> `Group=www-data`                                           |

---

## 🛡️ Security Best Practices

- Never run services like Gunicorn as `root`.
- Ensure Nginx only has **read access** to static and media files.
- Use strong permissions: `deployuser` writes, `www-data` reads.
- Log and monitor access to these directories if needed.


## ✅ Summary

| Component | Purpose |
|----------|---------|
| **Gunicorn** | Runs the Django app as a WSGI server |
| **Nginx** | Acts as a reverse proxy, serves static files |
| **Systemd** | Keeps Gunicorn running and restarts on failure |
| **Collectstatic** | Copies static files to `STATIC_ROOT` for Nginx to serve |
| **alias** in Nginx | Maps URL to file path |
| **expires / cache-control** | Improves performance via client-side caching |

---

Happy deploying! 🎉
