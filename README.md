# Configuring-Linux-Servers
Udacity Project 6

## Description

You will take a baseline installation of a Linux server and prepare it to host your web applications. You will secure your server from a number of attack vectors, install and configure a database server, and deploy one of your existing web applications onto it.

The application meant to be deployed is the **Item catalog app**, previously developed for [Project 4](https://github.com/yrazlik/Item-Catalog-App).

## Project Info

IP address: 35.157.34.238

Accessible SSH port: 2200.

Application URL: http://35.157.34.238

## Walkthrough

### 1 - Create a new user named *grader* and give sudo permissions.

1. Log into the the server with ubuntu user `$ ssh ubuntu@35.157.34.238`.
2. Create a new user called *grader*: `$ sudo adduser grader`.
3. Create a new file under the directory: `$ sudo nano /etc/sudoers.d/grader`. Fill that newly created file with the following line of text: "grader ALL=(ALL) NOPASSWD:ALL", then save it.

### 2 - Update packages

1. `$ sudo apt-get update`.
2. `$ sudo apt-get upgrade`.

### 3 - Configure the local timezone to UTC

1. Open time configuration dialog and set it to UTC with: `$ sudo timedatectl set-timezone UTC`.

### 4 - Configure the key-based authentication for *grader* user

1. Generate an encryption key **on your local machine** named graderkey.pem with: `$ ssh-keygen`.
2. Log into the remote VM as *root* user through ssh and create the following file: `$ touch /home/grader/.ssh/authorized_keys`.
3. Copy the content of the *udacity_key.pub* file from your local machine to the */home/grader/.ssh/authorized_keys* file you just created on the remote VM. Then change some permissions:
	1. `$ sudo chmod 700 /home/grader/.ssh`.
	2. `$ sudo chmod 644 /home/grader/.ssh/authorized_keys`.
	3. Finally change the owner from *root* to *grader*: `$ sudo chown -R grader:grader /home/grader/.ssh`.

### 5 - Enforce key-based authentication
1. Edit `$ sudo nano /etc/ssh/sshd_config`. Set *PasswordAuthentication* line and edit it to *no*.
2. `$ sudo service ssh restart`.

### 6 - Change the SSH port from 22 to 2200
1. Edit `$ sudo nano /etc/ssh/sshd_config` file. Find the *Port* line and edit it to *2200*.
2. `$ sudo service ssh restart`.
3. Now you are able to log into the remote VM through ssh with the following command: `ssh -i graderkey.pem grader@35.157.34.238 -p 2200`.

### 7 - Disable ssh login for *root* user
1. Edit `$ sudo nano /etc/ssh/sshd_config` file. Find the *PermitRootLogin* line and edit it to *no*.
2. `$ sudo service ssh restart`.

### 8 - Configure the Uncomplicated Firewall (UFW)
1. `$ sudo ufw allow 2200/tcp`.
2. `$ sudo ufw allow 80/tcp`.
3. `$ sudo ufw allow 123/udp`.
4. `$ sudo ufw enable`.

### 9 - Install Apache, mod_wsgi

1. `$ sudo apt-get install apache2`.
2. Enable *mod_wsgi*: `$ sudo a2enmod wsgi`.
3. `$ sudo service apache2 start`.

### 10 - Install Git

1. `$ sudo apt-get install git`.
2. Configure your username: `$ git config --global user.name <username>`.
3. Configure your email: `$ git config --global user.email <email>`.

### 11 - Clone the Catalog app from Github

1. `$ cd /var/www`. Then: `$ sudo mkdir catalog`.
2. Change owner for the *catalog* folder: `$ sudo chown -R grader:grader catalog`.
3. Move inside that newly created folder: `$ cd /catalog` and clone the catalog repository from Github: `$ git clone https://github.com/yrazlik/Item-Catalog-App.git`.
4. Create a *catalog.wsgi* file ib/var/www/catalog directory to serve the application over the *mod_wsgi*. That file should look like this:

```python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
```

### 12 - Install project dependencies

1. Install *pip*: `$ sudo apt-get install python-pip`.
2. Install virtualenv: `$ sudo pip install virtualenv`.
3. cd into *catalog* folder: `$ cd /var/www/catalog`. Then create a new virtual environment with the following command: `$ sudo virtualenv venv`.
4. Activate the virtual environment: `$ source venv/bin/activate`.
5. Change permissions on this folder: `$ sudo chmod -R 777 venv`.
6. Install Flask: `$ pip install Flask`.
7. Install all the other project's dependencies: `$ pip install bleach httplib2 request oauth2client sqlalchemy python-psycopg2`. 

### 13 - Configure and enable a new virtual host

1. Create a virtual host config file: `$ sudo nano /etc/apache2/sites-available/catalog.conf`.
2. File content should look like this:
```
<VirtualHost *:80>
    ServerName 35.157.34.238
    ServerAlias alias
    ServerAdmin ubuntu@35.157.34.238
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

3. Enable the new virtual host: `$ sudo a2ensite catalog`.

### 14 - Install PostgreSQL

1. `$ sudo apt-get install libpq-dev python-dev`.
2. `$ sudo apt-get install postgresql postgresql-contrib`.
3. Create a new user called 'catalog' with his password: `# CREATE USER catalog WITH PASSWORD 'password';`.
4. Give *catalog* user the CREATEDB capability: `# ALTER USER catalog CREATEDB;`.
5. Create the 'catalog' database owned by *catalog* user: `# CREATE DATABASE catalog WITH OWNER catalog;`.
6. Connect to the database: `# \c catalog`.
7. Revoke all rights: `# REVOKE ALL ON SCHEMA public FROM public;`.
8. Lock down the permissions to only let *catalog* role create tables: `# GRANT ALL ON SCHEMA public TO catalog;`.
90. Log out from PostgreSQL: `# \q`. Then return to the *grader* user: `$ exit`.
10. Create db engine in your app.py file: 
```python
engine = create_engine('postgresql://catalog:sillypassword@localhost/catalog')
```
12. Setup the database with: `$ python /var/www/catalog/catalog/setup_database.py`.
13. Do not allow remote connections to the database. Open the file: `$ sudo nano /etc/postgresql/9.3/main/pg_hba.conf` and edit it. It should look like this: 
```
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```

### 19 - Restart Apache to launch the app
1. `$ sudo service apache2 restart`.

#### Special thanks to [*stueken*](https://github.com/stueken) for his helpful README in his [repository](https://github.com/stueken/FSND-P5_Linux-Server-Configuration).
