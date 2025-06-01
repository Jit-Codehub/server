# How to Host Django Application using Gunicorn & Nginx in Production

---

### Step 1 - Installing Python and Nginx

```bash
sudo apt update
sudo apt install python3-pip python3-dev nginx
```

This will install Python, pip, and the nginx server.

---

### Step 2 - Creating a Python Virtual Environment

```bash
sudo apt install python3-venv
python3 -m venv venv
source venv/bin/activate
```

---

### Step 3 - Installing Django and Gunicorn

```bash
pip install django gunicorn
```

This installs Django and gunicorn in our virtual environment.

---

### Step 4 - Configuring Django

Add your IP address or domain to the `ALLOWED_HOSTS` variable in `settings.py`.

If you have any migrations to run:

```bash
python manage.py makemigrations
python manage.py migrate
```

Test the sample project by running the following commands:

```bash
sudo ufw allow 8000
python manage.py runserver 0.0.0.0:8000
```

You can now access your app at `http://<your-ip>:8000`.

---

### Step 5 - Configuring Gunicorn

```bash
gunicorn --bind 0.0.0.0:8000 folder_name_in_which_wsgi_file_is_located.wsgi
```

This should start Gunicorn on port 8000. Visit `http://<ip-address>:8000` to check your app.

Deactivate the virtual environment (optional):

```bash
deactivate
```

---

### Step 6 - Create a Systemd Socket File for Gunicorn

```bash
sudo vim /etc/systemd/system/gunicorn.socket
```

Paste the following:

```ini
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```

---

### Step 7 - Create a Gunicorn Service File

```bash
sudo vim /etc/systemd/system/gunicorn.service
```

Paste the following and update paths accordingly:

```ini
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=root
Group=www-data
WorkingDirectory=/compelte_folder_path_inside_which_django_project_is_located
ExecStart=/complete_viruatlenv_path/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          folder_name_in_which_wsgi_file_is_present.wsgi:application
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Example file of gunicorn.service

```ini
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=root
Group=www-data
WorkingDirectory=/home/equity_master_project  #directory in which django project is present
ExecStart=/home/equity_master_project/venv/bin/gunicorn \           ##virtualenv path to activate gunicorn
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          folder_name_where_wsgi_file_is_present.wsgi:application   #update wsgi file name in this line
Restart=always
RestartSec=3
[Install]
WantedBy=multi-user.target
```

---

### Step 8 - Start and Enable Gunicorn Socket

```bash
sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket
```

---

### Step 9 - Configuring Nginx as a Reverse Proxy

```bash
sudo vim /etc/nginx/sites-available/project_name
```

Paste the following:

```nginx
server {
    listen 80;
    server_name 88.222.242.103;  # Use your IP address or domain

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /home/equity_master_project;  # Path to your Django static files
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
```

Activate the configuration:

```bash
sudo ln -s /etc/nginx/sites-available/project_name /etc/nginx/sites-enabled/
```

---

### Step 10 - Restart and Enable Services

```bash
sudo systemctl restart nginx
sudo systemctl daemon-reload
sudo systemctl enable gunicorn.service
sudo systemctl start gunicorn.service

sudo systemctl restart gunicorn
sudo systemctl status gunicorn
```

---

You're now successfully hosting a Django app with Gunicorn and Nginx in production!
