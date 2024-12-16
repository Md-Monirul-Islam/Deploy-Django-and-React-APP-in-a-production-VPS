# Deploy Django and React App in Production Environment

This guide walks you through deploying a Django and React application on a production VPS with Ubuntu Server. The setup uses PostgreSQL as the database, NGINX as the reverse proxy, and assumes the following directory structure:

- **Backend Code**: `/home/backend/`
- **Frontend Code**: `/home/frontend/`

## Prerequisites

1. A VPS with Ubuntu Server installed.
2. Git installed for uploading your code.

---

## Steps to Deploy

### 1. Install Required Packages

Update the Ubuntu package index and install necessary dependencies:

```bash
sudo apt-get update
sudo apt-get install python3-pip python3-dev libpq-dev postgresql postgresql-contrib nginx
```

### 2. Setup the Backend

Navigate to your backend folder:

```bash
cd /home/backend/
```

---

### 3. Create PostgreSQL Database and User

1. Enter the PostgreSQL environment:

   ```bash
   sudo -u postgres psql
   ```

2. Create a database and user with a password, and assign the user to the database. Replace `db_name`, `db_user_name`, and `db_user_password` with your custom values:

   ```sql
   CREATE DATABASE db_name;
   CREATE USER db_user_name WITH PASSWORD 'db_user_password';
   GRANT ALL PRIVILEGES ON DATABASE db_name TO db_user_name;
   \q
   ```

---

### 4. Create a Python Virtual Environment

Upgrade `pip` and install `virtualenv`:

```bash
sudo -H pip3 install --upgrade pip
sudo -H pip3 install virtualenv
```

Create a virtual environment named `venv`:

```bash
virtualenv venv
```

Activate the virtual environment:

```bash
source venv/bin/activate
```

Use `pip freeze > requirements.txt` in your local development environment to export all required package names with their versions. Then, install the required packages:

```bash
pip install -r requirements.txt
```

For GCP VMs, using `pip` without `sudo` might cause permission errors. In such cases, run the following command:

```bash
sudo /home/backend/venv/bin/python -m pip install -r requirements.txt
```

---

### 5. Install Additional Dependencies

Install Gunicorn (a Python WSGI HTTP Server) and `psycopg2` (PostgreSQL adapter for Python):

```bash
pip install gunicorn psycopg2
```

You should now have everything you need to deploy your Django backend.

---

### 6. Migrate Your Database

Apply your database migrations:

```bash
python manage.py migrate
```

---

### 7. Test Your Backend

Verify that the project starts successfully:

```bash
python manage.py runserver
```

If you encounter any module not found errors or other issues, resolve them before moving to the next step.

Create an exception for port 8000:

```bash
sudo ufw allow 8000
```

Test your project by starting the Django development server:

```bash
python manage.py runserver 0.0.0.0:8000
```

In your web browser, visit your server’s domain name or IP address followed by `:8000`:

```text
http://server_domain_or_IP:8000
```

If your project is running successfully, proceed to the next step. Otherwise, debug any issues before continuing.

---

### 8. Test Gunicorn’s Ability to Serve the Project

Replace `project_name` with your project folder name (where your `urls.py` and `wsgi.py` files exist):

```bash
gunicorn --bind 0.0.0.0:8000 project_name.wsgi
```


This will start Gunicorn on the same interface that the Django development server was running on. You can go back and test the app again.

**Note:** The admin interface will not have any of the styling applied since Gunicorn does not know about the static CSS content responsible for this.

If everything so far has gone well, deactivate the virtual environment:

```bash
deactivate
```

The virtual environment indicator in your terminal will be removed.

---

### 9. Create a Gunicorn systemd Service File

We have tested that Gunicorn can interact with our Django application, but we should implement a more robust way of starting and stopping the application server. To accomplish this, we’ll make a systemd service file.

Create and open a systemd service file for Gunicorn with sudo privileges:

```bash
sudo nano /etc/systemd/system/gunicorn.service
```

Paste the following code in the editor. Replace `root` with your username and `project_name` with your project folder name:

```ini
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=root
Group=www-data
WorkingDirectory=/home/backend
ExecStart=/home/backend/venv/bin/gunicorn \
  --env DJANGO_SETTINGS_MODULE=project_name.settings \
  --access-logfile /home/backend/logs/gunicorn.log \
  --workers 3 --bind 127.0.0.1:8000 project_name.wsgi:application

[Install]
WantedBy=multi-user.target
```

Now press `Ctrl + X` to exit the editor, type `Y` to confirm changes, and press `Enter` to save.

---

### 10. Start and Enable Gunicorn Service

Start the Gunicorn service that we created and enable it to start at boot:

```bash
sudo systemctl start gunicorn
sudo systemctl enable gunicorn
```

### 11. Check Gunicorn Status and Logs

Check the status of the Gunicorn service to verify that it started successfully:

```bash
sudo systemctl status gunicorn
```

If the service started correctly, the output will indicate that the service is active and running. Next, check the contents of your backend directory to ensure that the socket file is present (backend.sock):

```bash
ls /home/backend
```

If you encounter any issues, or if the socket file is missing, inspect the Gunicorn process logs:

```bash
sudo journalctl -u gunicorn
```

Take a look at the messages in the logs to find out where Gunicorn ran into problems. There are many reasons that you may have run into problems, but often, if Gunicorn was unable to create the socket file, it is for one of these reasons:

1. The project files are owned by the root user instead of a sudo user.
2. The `WorkingDirectory` path within the `/etc/systemd/system/gunicorn.service` file does not point to the project directory.
3. The configuration options given to the Gunicorn process in the `ExecStart` directive are not correct. Check the following items:
   - The path to the Gunicorn binary points to the actual location of the binary within the virtual environment.
   - The `--bind` directive defines a file to create within a directory that Gunicorn can access.
   - The `project_name.wsgi:application` is an accurate path to the WSGI callable.

If you make changes to the `/etc/systemd/system/gunicorn.service` file, reload the daemon to re-read the service definition and restart the Gunicorn process by typing:

```bash
sudo systemctl daemon-reload
sudo systemctl restart gunicorn
```

Make sure you troubleshoot any of the above issues before continuing.

---

### 12. Configure NGINX for Reverse Proxy to Gunicorn

Check existing enabled sites by:

```bash
ls /etc/nginx/sites-enabled/
```

You'll see a file called `default`. If there are any other files, remove them:

```bash
sudo rm /etc/nginx/sites-enabled/filename
```

Check available servers:

```bash
ls /etc/nginx/sites-available/
```

You'll see a file called `default`. If there are any other files, remove them as well. Backup the default server file:

```bash
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/default.bak
```

Now open that default server definition file:

```bash
sudo nano /etc/nginx/sites-available/default
```


Paste the following block of code:

```bash
server {
    root /home/frontend/build;
    index index.htm index.html index.nginx-debian.html;
    server_name 1**.**.**.*9;

    location / {
        try_files $uri $uri/ /index.html =404;
    }

    location ~ ^/api {
        proxy_pass http://localhost:8000;
        proxy_set_header HOST $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name 1**.**.**.*9;
    return 404;
}
```

Change `server_name` value with your domain or IP address.

**Note:** Look at that root, we defined `/home/frontend/build` where our frontend codes will be available.



Test your Nginx configuration for syntax errors by typing:
   ```bash
   sudo nginx -t
   ```
If no errors are reported, go ahead and restart Nginx by typing:
   ```bash
   sudo systemctl restart nginx
   ```

Finally, we need to open up our firewall to normal traffic on port 80. Since we no longer need access to the development server, we can remove the rule to open port 8000 as well:

   ```bash
   sudo ufw delete allow 8000
   sudo ufw allow 'Nginx Full'
   ```
And our backend is ready