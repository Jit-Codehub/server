# Best Practices for Hosting Django with Gunicorn and Nginx

This guide explains the most important best practices when deploying a Django application using Gunicorn and Nginx in production. It covers caching, security, user permissions, and monitoring — everything you need for a secure and performant deployment.

---

## 1. Use `expires` Headers and Caching Rules in Nginx for Static Files

### Why?

Static files (like images, CSS, and JavaScript) don't change frequently. Caching them:

* Reduces server load
* Speeds up site loading
* Improves overall performance

### How?

Edit your Nginx config (e.g. `/etc/nginx/sites-available/your_project`) and add:

```nginx
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
```

### Important:

* If a file changes before the caching duration ends, browsers may still load the old version.
* To solve this, use **cache busting** (Django does this with `ManifestStaticFilesStorage`).

---

## 2. Limit Access Permissions for Static and Media Directories

### Why?

* Restricting permissions prevents accidental or malicious changes.
* Ensures that only Nginx (or the correct user) can access or read those files.

### How?

Set the correct ownership and permission:

```bash
sudo chown -R www-data:www-data /home/equity_master_project/staticfiles
sudo chown -R www-data:www-data /home/equity_master_project/media

sudo chmod -R 755 /home/equity_master_project/staticfiles
sudo chmod -R 755 /home/equity_master_project/media
```

### Recommendations:

* Avoid giving `777` permissions.
* Never let user uploads be world-writable.
* For private media, serve through Django views with authentication.

---

## 3. Run Gunicorn Under a Dedicated Non-root User

### Why?

* Running as root is dangerous. If compromised, attackers gain full control.
* Running with a dedicated user limits potential damage.

### How?

1. **Create a system user**:

```bash
sudo useradd --system --group www-data --home /home/equity_master_project equityuser
```

2. **Change ownership of the project**:

```bash
sudo chown -R equityuser:www-data /home/equity_master_project
```

3. **Update your Gunicorn systemd service file**:

```ini
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=equityuser
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

4. **Reload and start Gunicorn**:

```bash
sudo systemctl daemon-reload
sudo systemctl start gunicorn
sudo systemctl enable gunicorn
```

---

## 4. Monitor and Log Nginx and Gunicorn Logs for Issues

### Why?

* Helps detect configuration issues
* Lets you troubleshoot user errors and app crashes
* Crucial for performance monitoring and security auditing

### Locations:

| Service  | Log Type    | Location                     |
| -------- | ----------- | ---------------------------- |
| Nginx    | Access Log  | `/var/log/nginx/access.log`  |
| Nginx    | Error Log   | `/var/log/nginx/error.log`   |
| Gunicorn | Systemd Log | Use `journalctl -u gunicorn` |

### Commands:

```bash
# View Nginx error logs
sudo tail -f /var/log/nginx/error.log

# View Gunicorn logs
sudo journalctl -u gunicorn -f

# Check Gunicorn status
sudo systemctl status gunicorn

# Restart Nginx
sudo systemctl restart nginx

# Check Nginx status
sudo systemctl status nginx
```

---

## Summary

| Task                         | Why It Matters                      |
| ---------------------------- | ----------------------------------- |
| Caching static/media files   | Speeds up user experience           |
| Setting correct permissions  | Secures file access                 |
| Running Gunicorn as non-root | Minimizes system compromise risks   |
| Logging and monitoring       | Detects, fixes, and prevents issues |

Follow these practices to ensure your Django application is secure, optimized, and production-ready.
