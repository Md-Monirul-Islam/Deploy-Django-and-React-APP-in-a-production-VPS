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