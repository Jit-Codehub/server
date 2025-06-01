# 🚀 How to Host Django Application using Gunicorn & Nginx in Production

This guide walks you through deploying a Django application using **Gunicorn** as the WSGI server and **Nginx** as a reverse proxy.

---

## 📦 Step 1 - Installing Python and Nginx

```bash
sudo apt update
sudo apt install python3-pip python3-dev nginx
```

This installs Python, pip, and the Nginx web server.

---

## 🐍 Step 2 - Creating a Python Virtual Environment

```bash
sudo apt install python3-venv
python3 -m venv venv
source venv/bin/activate
```

This creates and activates a Python virtual environment.

---

## 🌐 Step 3 - Installing Django and Gunicorn

```bash
pip install django gunicorn
```

This installs Django and Gunicorn into the virtual environment.

---

## 🛠 Step 4 - Django Settings & Migration

In your `settings.py`:

* Add your IP or domain to `ALLOWED_HOSTS`:

  ```python
  ALLOWED_HOSTS = ['your_ip_or_domain']
  ```
* Set `DEBUG = False`

Run migrations:

```bash
python manage.py makemigrations
python manage.py migrate
```

To test the server:

```bash
sudo ufw allow 8000
python manage.py runserver 0.0.0.0:8000
```

Visit `http://your_ip:8000` in a browser to verify.

---

## 🔧 Step 5 - Configuring Gunicorn

Run Gunicorn manually to test:

```bash
gunicorn --bind 0.0.0.0:8000 folder_name_where_wsgi_file_is_located.wsgi
```

To run Gunicorn as a service:

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
User=root
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

## 🌐 Step 6 - Configuring Nginx as a Reverse Proxy

```bash
sudo vim /etc/nginx/sites-available/equity
```

Paste:

```nginx
server {
    listen 80;
    server_name 88.222.242.103;

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

> 📘 We use `alias` with absolute paths so that Nginx serves these files directly, instead of Django, improving performance. Django should serve dynamic content only.

Enable config:

```bash
sudo ln -s /etc/nginx/sites-available/equity /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl restart gunicorn
```

---

## 🌐 Step 7 - Configure Static & Media Handling in Django

In `settings.py`:

```python
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')

MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
```

Then collect static files:

```bash
python manage.py collectstatic
```

This moves all static assets into `/home/equity_master_project/staticfiles/` which Nginx can serve directly.

---

## 🔐 Best Practices

### 🕒 Expires Headers

Improves browser caching:

```nginx
location /static/ {
    alias /home/equity_master_project/staticfiles/;
    expires 30d;
    add_header Cache-Control "public, max-age=2592000";
}
```

If a file changes before 30 days, browsers may still show old content unless you version your file names.

### 🔐 Limit Permissions

Ensure static/media directories are readable but not writable by the web process. Use:

```bash
sudo chown -R root:www-data /home/equity_master_project/staticfiles/
sudo chmod -R 755 /home/equity_master_project/staticfiles/
```

### 🧑‍💻 Use Non-Root Gunicorn User

Instead of `User=root`, use:

```ini
User=django
Group=www-data
```

Create user:

```bash
sudo adduser --system --group django
```

### 📊 Monitor Logs

Check logs regularly:

```bash
sudo journalctl -u gunicorn
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log
```

---

🎉 You're now ready for production!
