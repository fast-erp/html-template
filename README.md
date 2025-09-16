<!-- *********** if any problem to dump the database, run the following command ******************** -->
<!-- >>>>-->   python -Xutf8 manage.py dumpdata <app>.<object> -o db_dump_<suffix>.json
# For use in development
1. install postgresql
2. install memurai/redis-server
3. run the memurai/redis-server
4. run the command for initialize celery: celery -A worker --pool=solo -l info (inside activated venv) 

# Before production Use
>> 1. remove > app.conf.worker_pool = 'solo' from erp.celery.py
>> 2. use command > celery -A erp worker --loglevel=info

# Deployment Guide
### Update the system and install dependency
```
sudo apt update
sudo apt install python3-venv python3-dev libpq-dev postgresql postgresql-contrib nginx curl redis-server
```
### Start and enable Redis:
```
sudo systemctl start redis-server
sudo systemctl enable redis-server
```
### Start and enable postgresql:
```
sudo systemctl start postgresql
sudo systemctl enable postgresql
```
### Login to postgresql with postgres user
```
sudo -u postgres psql
```
### Create a database for your project:
```
CREATE DATABASE skg5;
```
### Create a database user for your project. Make sure to select a secure password:
```
CREATE USER erpadmin WITH PASSWORD 'SKG!@#$REWQ1234rewq';
```
### Set client_encoding to 'utf8', default transaction isolation scheme to “read committed” and timezone
```
ALTER ROLE erpadmin SET client_encoding TO 'utf8';
ALTER ROLE erpadmin SET default_transaction_isolation TO 'read committed';
ALTER ROLE erpadmin SET timezone TO 'UTC';
```
### Give the new user access to administer the new database:
```
GRANT ALL PRIVILEGES ON DATABASE skg5 TO erpadmin;
GRANT ALL ON SCHEMA public TO erpadmin;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO erpadmin;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO erpadmin;
```
### Exit out of the PostgreSQL prompt by typing:

```
\q
```
### Make a new directory and change directory to that directory
```
mkdir erpdir
cd erpdir
```
### Clone the github repository on current directory:
```
git clone https://github.com/fast-erp/fast-erp.git .
```
### Create a Python virtual environment by typing:
```
python3 -m venv venv
```
### Activate the virtual environment:
```
source venv/bin/activate
```
### Upgrade the pip and Install required packages by typing:
```
python -m pip install --upgrade pip
pip install -r requirements.txt
```
### Set the hostnames on Allowed host
```
sudo nano /erpdir/erp/settings.py
. . .
# The simplest case: just add the domain name(s) and IP addresses of your Django server
# ALLOWED_HOSTS = [ 'example.com', '203.0.113.5']
# To respond to 'example.com' and any subdomains, start the domain with a dot
# ALLOWED_HOSTS = ['.example.com', '203.0.113.5']
ALLOWED_HOSTS = ['your_server_domain_or_IP', 'second_domain_or_IP', . . ., 'localhost']
```
### Change the settings with your PostgreSQL database information (if needed):
```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'myproject',
        'USER': 'myprojectuser',
        'PASSWORD': 'password',
        'HOST': 'localhost',
        'PORT': '',
    }
}
```
### MakeMigration and Migrate the Database
```
/erpdir/manage.py makemigrations
/erpdir/manage.py migrate
```
### Create an administrative user for the project by typing:
```
/erpdir/manage.py createsuperuser
```
### Collect Static Files
```
/erpdir/manage.py collectstatic
```
### Create an exception for port 8000 by typing:
```
sudo ufw allow 8000
```
### Finally, you can test out your project by starting up the Django development server with this command:
```
/erpdir/manage.py runserver 0.0.0.0:8000
```
### In your web browser, visit your server’s domain name or IP address followed by :8000:
```
http://server_domain_or_IP:8000
```
### Test the ability of gunicorn to serve
```
cd /erpdir
gunicorn --bind 0.0.0.0:8000 myproject.wsgi
```
### deactivate the virtual env and create gunicorn socket file
```
deactivate
sudo nano /etc/systemd/system/gunicorn.socket
```
>[Unit]
>Description=gunicorn socket

>[Socket]
>ListenStream=/run/gunicorn.sock
>
>[Install]
>WantedBy=sockets.target

### Create the gunicorn service file
```
sudo nano /etc/systemd/system/gunicorn.service
```
>[Unit]
>Description=gunicorn daemon
>Requires=gunicorn.socket
>After=network.target
>
>[Service]
>User=username_of_server
>Group=www-data
>WorkingDirectory=/home/username_of_server/erpdir
>ExecStart=/home/username_of_server/erpdir/venv/bin/gunicorn \
>          --access-logfile - \
>          --workers 3 \
>          --bind unix:/run/gunicorn.sock \
>          erp.wsgi:application
>
>[Install]
>WantedBy=multi-user.target

### Start and enable and check the status of gunicorn service
```
sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket
sudo systemctl status gunicorn.socket
```
output should be:
>
>Output
>● gunicorn.socket - gunicorn socket
>     Loaded: loaded (/etc/systemd/system/gunicorn.socket; enabled; vendor preset: enabled)
>     Active: active (listening) since Mon 2022-04-18 17:53:25 UTC; 5s ago
>   Triggers: ● gunicorn.service
>     Listen: /run/gunicorn.sock (Stream)
>     CGroup: /system.slice/gunicorn.socket
>
>Apr 18 17:53:25 django systemd[1]: Listening on gunicorn socket.
>

### Check the existense of gunicorn soc file:
```
file /run/gunicorn.sock
```
> output: /run/gunicorn.sock: socket
### Now, reload daemon and restart gunicorn
```
sudo systemctl daemon-reload
sudo systemctl restart gunicorn
```
## Configure NGINX:
### Configure nginx
```
sudo nano /etc/nginx/sites-available/erp
```
>server {
>    listen 80;
>    server_name server_domain_or_IP;
>
>    location = /favicon.ico { access_log off; log_not_found off; }
>    location /static/ {
>        root /home/sammy/myprojectdir;
>    }
>
>    location / {
>        include proxy_params;
>        proxy_pass http://unix:/run/gunicorn.sock;
>    }
>}

### Enable the file by linking it to the sites-enabled directory, test and restart the nginx:
```
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled
sudo nginx -t
```
### Change the ownership of the directory to others sothat nginx can access and serve static and media assets:
```
chmod o+x /home/web-sm
```
### Remove port 8000 and enable nginx on ufw:
```
sudo ufw delete allow 8000
sudo ufw allow 'Nginx Full'
```
### Restart everything and test the deployment:
```
sudo systemctl daemon-reload
sudo systemctl restart gunicorn
sudo systemctl restart nginx
```
### If make any changes in gunicorn socket or gunicorn service file, reload those:
```
sudo systemctl restart gunicorn.socket gunicorn.service
```
### If you change the Nginx server block configuration, test the configuration and then Nginx by typing:
```
sudo nginx -t && sudo systemctl restart nginx
```

# Intall Certbot for free SSL certificate
```
sudo apt install certbot python3-certbot-nginx -y
```
## Update Your Nginx Config with Domain Names
### Change server_name in your config from the IP to:
```
server_name skgbd.com www.skgbd.com;
```
### Then reload Nginx to apply changes:
```
sudo nginx -t && sudo systemctl reload nginx
```
## Obtain and Install SSL Certificate
### Run Certbot for Nginx:
```
sudo certbot --nginx -d skgbd.com -d www.skgbd.com
```
### Auto-Renewal (Recommended)
```
sudo certbot renew --dry-run
```
