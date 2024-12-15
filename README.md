Deploy Django and React APP in a production VPS (Django + React + PostgreSQL + NGINX + Ubuntu Server)

Assuming You have backend and frontend codes in /home/backend and /home/frontend/ (Use git to upload)

1. **Install required Packages from the Ubuntu Repositories
    ```bash
    sudo apt-get update
    sudo apt-get install python3-pip python3-dev libpq-dev postgresql postgresql-contrib nginx

