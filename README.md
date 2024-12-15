Deploy Django and React APP in a production VPS (Django + React + PostgreSQL + NGINX + Ubuntu Server)

Assuming You have backend and frontend codes in /home/backend and /home/frontend/ (Use git to upload)

1. **Install required Packages from the Ubuntu Repositories**
    ```bash
    sudo apt-get update
    sudo apt-get install python3-pip python3-dev libpq-dev postgresql postgresql-contrib nginx

2. **Setup Backend**
We'll setup backend first. Go to your backend folder
    ```bash
    cd /home/backend/

3. **Create PostgreSQL Database and User**
Enter postgres environment
    ```bash
    sudo -u postgres psql

Create database and user with password and assign that user to that database. Change the db_name, db_user_name and db_user_password with yours.
    ```bash
    CREATE DATABASE db_name;
    CREATE USER db_user_name WITH PASSWORD 'db_user_password';
    GRANT ALL PRIVILEGES ON DATABASE db_name TO db_user_name;
    \q