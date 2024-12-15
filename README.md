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
---

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

Install gunicorn and psycopg2 (for postgresql) additionally
    ```bash
    pip install gunicorn psycopg2
    ```
You should now have everything you need to deploy your django backend.

Migrate your database migrations

    ```bash
    python manage.py migrate
    ```

Now try if you're able to start the project.
    ```bash
    python manage.py runserver
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