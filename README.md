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

---

### 13. Setup Frontend
Move to your frontend folder
   ```bash
   cd /home/frontend
   ```
Install nodejs and npm
   ```bash
   sudo apt-get install nodejs npm
   ```

Install required libraries.
   ```bash
   sudo npm install
   ```
Build the project
   ```bash
   sudo npm run build
   ```

Restart nginx service
   ```bash
   sudo service nginx restart
   ```
You should now be able to go to your server’s domain or IP address to view your application.

---

### 14. BONUS: Access Django Admin Area
By previous settings you won't be able to access django-admin. If you want to access it, along with styling (Gunicorn will not load any styling), change your nginx configuration file as bellow.

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
    
    location ~ ^/admin {
        proxy_pass http://localhost:8000;
        proxy_set_header HOST $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location = /favicon.ico {
        access_log off;
        log_not_found off;
    }
    
    location /static {
        autoindex on;
        alias /home/backend/staticfiles;
    }

    location /media {
        autoindex on;
        alias /home/backend/media;
    }
}

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name 1**.**.**.*9;
    return 404;
}
```

In your django settings file, add this.
If already exist then no need to configure it.

   ```bash
   STATIC_URL = "/static/"
   STATIC_ROOT = "staticfiles"
   ```
**Note:** Note: If your django admin lives in other url than default /admin/ change that url here location ~ ^/admin.

Now run this management command to collect all static files to that folder (staticfiles).

   ```bash
   python manage.py collectstatic
   ```

Now you should access your django admin area along with styling (CSS & JS).
   ```bash
   http://server_domain_or_IP/admin
   ```
### Bonus
As you update your configuration or application, you will likely need to restart the processes to adjust to your changes.
If you update your Django application, you can restart the Gunicorn process to pick up the changes by typing:

   ```bash
   sudo systemctl restart gunicorn
   ```

If you change gunicorn systemd service file, reload the daemon and restart the process by typing:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart gunicorn
   ```

If you change the Nginx server block configuration, test the configuration and then restart Nginx by typing:

   ```bash
   sudo nginx -t && sudo systemctl restart nginx
   ```

These commands are helpful for picking up changes as you adjust your configuration.


**Note:** Comment bellow if you face any error while deploying your application, I'll try to reply. Don't forget to put down your application url in comment box after you successfully deloyed it.

### 15. Django, Celery, Redis in Production using Supervisor
Django, Celery, Redis in Production using Supervisor.md

## What is Celery?

Celery is an **asynchronous** **task** queue/job queue based on distributed message passing. It is focused on real-time operation but supports **scheduling** as well.

**Why is this useful?**

*   Think of all the times you have had to run a certain task in the future. Perhaps you needed to access an API every hour. Or maybe you needed to send a batch of emails at the end of the day. Large or small, Celery makes scheduling such periodic tasks easy.
*   You never want end users to have to wait unnecessarily for pages to load or actions to complete. If a long process is part of your application’s workflow, you can use Celery to execute that process in the background, as resources become available, so that your application can continue to respond to client requests. This keeps the task out of the application’s context.

# What is Redis?

**Redis** is an in-memory data structure project implementing a distributed, in-memory key-value database with optional durability.

The most common Redis use cases are **session cache, full-page cache, queues, leaderboards and counting, publish-subscribe**, and much more. in this case, we will use Redis as a **message broker.**

> Celery uses “[brokers](http://celery.readthedocs.org/en/latest/getting-started/brokers/)” to pass messages between a Django Project and the Celery [workers](http://celery.readthedocs.org/en/latest/userguide/workers.html).

**For more details visit** [**Django**](https://docs.djangoproject.com/en/3.0/)**,** [**Celery**](https://docs.celeryproject.org/en/stable/userguide/)**, and** [**Redis**](https://redis.io/documentation) **official documentation.**

# **Celery Implementation with Django Step by Step:**

## Step 1. Create a Django Application.

First, you need to create a Django application. I am not describing how to do that, you can easily understand those things from Django documentation or I am suggesting you this [**site**](https://tutorial.djangogirls.org/en/) for the basic creation process of Django application.

## Step 2: Install Celery Using pip.

Before installing celery please activate your **virtual** **environment,** then run the command from the terminal or other command-line like git-bash:

```bash
pip install celery
```
## Step 3. Add celery.py File in Your Project Module.

celery.py

```bash
import os  
from celery import Celery  

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'your_project_name.settings')

app = Celery('your_project_name')

app.config_from_object('django.conf:settings', namespace="CELERY")
app.autodiscover_tasks()

@app.task(bind=True)  
def debug_task(self):  
    print('Request: {0!r}'.format(self.request))
```

> please, make sure to change the **your_project_name** with your actual project name that you created

## Step 4: Import the Celery App to Django.

To ensure that the Celery app is loaded when Django starts, add the following code into the **__init__.py** file that sits on the project module beside on settings.py file.

```bash
from .celery import app as celery_app

__all__ = ['celery_app']
```

## Step 5: Download Redis as a Celery ‘broker’.

First install [Redis](http://redis.io/)
```bash
sudo apt install redis
```
If you're an windows user, download it from here. https://github.com/microsoftarchive/redis/releases/

in a new terminal window, fire up the server with this command:
```bash
redis-server
```

You can test that Redis is working properly by typing this into your terminal:
```bash
redis-cli ping
```

> Redis should reply with `PONG`!

## Step 6: Add Redis as a Dependency in the Django Project:

Run the command:

```bash
pip install redis
```

## step 7: Celery Stuff Configure to the Django Settings File

Once Redis is up, add the following code to your settings.py file:

```bash
CELERY_BROKER_URL = "redis://localhost:6379"
CELERY_RESULT_BACKEND = 'django-db'
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
```

## That’s it! You should now be able to use Celery with Django.

Test that the Celery worker is ready to receive tasks:

```bash
celery -A your_project_name worker -l info
```

Modify `**your_project_name**` with the actual name of your project

> **The most important task is: Always run a worker to execute the celery task**

# Add a New Task to the Celery:

## Step 1: Add *tasks.py* File to Your Django App.

```bash
from celery import shared_task
from time import sleep

@shared_task
def my_first_task(duration):
    sleep(duration)  
    return('task_done')
```

Your new task is ready to be used but you have to assign it to the celery.

## Step 2: Assign Task to the Celery.

You need to assign a task to the celery. To assign this task you need to call this function with something different. celery gives us two methods **delay()** and **apply_async()** to call task.

```bash
celery -A **your_project_name** worker -l info
```

> please modify `**your_project_name**` with the actual name of your project

you will see a new task **my_first_task** is assigned in celery


```bash
[tasks]  
 . your_project_name.celery.debug_task
 . my_first_task
```

# Let's complete and test the task with the response.

## Create a View in your App

```bash
from django.http import HttpResponse
from .tasks import my_first_task

def index(request):  
    my_first_task.delay(10)
    return HttpResponse('response done')
```

**my_first_task** is a function that I assigned in **tasks.py** file, and I used to **delay()** to assign a task in celery worker.

## Call the view from your app URL.

```bash
from django.urls import path
from .views import index

urlpatterns = [  
    path('celery-test/',index, name='celery_test_url'),  
]
```

when you make a request to this URL you will see its response to you **‘response done’** just within 2 or 3 seconds but what really should have happened? I passed **10** to **my_first_task** function as an argument with the delay() method. so the **duration** parameter will take this **10** and will call **sleep**(10) so this function should wait for 10 seconds to return but it didn’t. The response back within too short!

> **This is the power of celery, you got the response in just time but the task is running by a worker in the background.**


# Celery In Production Using Supervisor on Linux Server

Running Celery locally is easy: simple **_celery -A your_project_name worker -l info_** does the trick. In production, though, something a little more robust is needed.

Thankfully, it’s not difficult at all. Just like Gunicorn, Celery can be overseen by Supervisor.

On a production server, the application needs to start and stop with the operating system. To achieve this, a _process control system_ is usually required (particularly for daemons that don’t do this job themselves, like Gunicorn’s). The supervisor is a particularly good example and is pretty simple to set up.

## Step 1: Install Supervisor on Ubuntu Server

```bash
sudo apt-get install supervisor
```

## Step 2: Add .conf File in Supervisor

```bash
sudo nano /etc/supervisor/conf.d/celery.conf
```

## Step 3: Add some Configure in celery.conf

```bash
[program:celery]
command=/path/to/env/bin/celery -A your_project_name worker -l info
directory=/path/to/workflow/your_project_name/
user=www-data
autostart=true
autorestart=true  
stdout_logfile=/path/to/workflow/your_project_name/logs/celery.log  
redirect_stderr=true
```

> **celery** can be anything you like, its totally up to you. this name is related to the supervisor, nothing else. in the future you able to access this app with this name

## Step 4: Inform Configuration to the Server

After adding a new program, we should run the following two commands, to inform the server to reread the configuration files and to apply any changes.

```bash
sudo supervisorctl reread  
sudo supervisorctl update
```

> Congratulation! You did all the tasks need, Now the Celery is ready to use in the production server!

# Managing Supervisor App

execute the `**supervisorctl**` client.

```bash
sudo supervisorctl
```

You will be greeted with a list of the registered processes. You will see a process called **your_app_name** with a `RUNNING` status.

```bash
your_app_name                 RUNNING   pid 6853, uptime 0:22:30  
supervisor>
Type `help` for a list of available commands.
```

We can `start`, `stop` and `restart` programs by passing the program name as an argument to the respective command.

We can also take a look at the program output with the `tail` command.

Once you are finished, you can quit.

```bash
supervisor> quit
```

# Running Celery Beat For Schedule Job
Install `django-celery-beat` and add to `INSTALLED_APPS`

```bash
pip install django-celery-beat
```

```bash
INSTALLED_APPS = [
    ...,
    "django_celery_beat",
]
```

## Add .conf File in Supervisor

```bash
sudo nano /etc/supervisor/conf.d/celery_beat.conf
```

```bash
[program:celery_beat]  
command=/path/to/env/bin/celery -A your_project_name beat --scheduler django -l info
directory=/path/to/workflow/your_project_name/  
user=www-data  
autostart=true  
autorestart=true
stdout_logfile=/path/to/workflow/your_project_name/logs/celery_beat.log  
redirect_stderr=true
```

## Reread conf files
```bash
sudo supervisorctl reread
sudo supervisorctl update
```

You will need to restart the worker everytime you change something

```bash
sudo supervisorctl restart celery
sudo supervisorctl restart celery_beat
```