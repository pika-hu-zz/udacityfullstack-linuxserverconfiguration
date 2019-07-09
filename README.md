# Linux Server Configuration

> Raymond Hu

## Overview

This file serves as documentation for the Linux Server Configuration project for Udacity's Fullstack Nanodegree program. 

The purpose of this project is to take a baseline installation of Linux on a virutal machine and prepare it to host web applications. The project involves securing the server from a number of attack vectors, installing and configuring a database server, and deploying the Aspire Volunteer Tracker onto it.

## Server Information

* Public IP Address: 35.175.151.242
* SSH Port: 2200
    * SSH into server using provided PEM private key and command ```ssh -i [local location of private key] -p 2200 grader@35.175.151.242```
* URL: http://35.175.151.242.xip.io/ (Use this URL for Goolge Federation)
* Software Installed: Python, Apache, Mog_wsgi, Git, VirtualVEnv, Flask, PostgreSQL, SQLAlchemy, Google OAuth, HTTPLib2, Psycopg2, Requests

## Configurations Made

1. Ensure currently installed packages are up-to-date and automatically update once a day
    * SSH into server using Amazon's browser SSH client (alternatively download Amazon's default private key and use SSH client of choice)
    * Update all currently installed packages: ```sudo apt-get update``` ```sudo apt-get upgrade```
    * Install unattended-upgrades package: ```sudo apt install unattended-upgrades```
    * Uncomment ```"${distro_id}:${distro_codename}-updates";``` from ```/etc/apt/apt.conf.d/50unattended-upgrades```
    * Edit ```/etc/apt/apt.conf.d/20auto-upgrades``` to the following config options:
         ```
         APT::Periodic::Update-Package-Lists "1";
         APT::Periodic::Download-Upgradeable-Packages "1";
         APT::Periodic::AutocleanInterval "7";
         APT::Periodic::Unattended-Upgrade "1";
         ```
2. Setup *grader* User
    * Create *grader*: ```sudo adduser grader```
    * Give *grader* sudo privileges: ```vi /etc/sudoers.d/grader``` and add ```grader ALL=(ALL:ALL) ALL`` to the newly created file
    * Setup SSH for grader:
        1. Switch to grader: ```sudo su grader```
        2. If not at home directory, cd to home directory
        3. Make .ssh directory: ```mkdir .ssh```
        4. Secure .ssh directory: ```chmod 700 .ssh```
        5. Create public key storage file: ```touch .ssh/authorized_keys```
        6. Secure public key storage: ```chmod 600 .ssh/authorized_keys```
        7. Copy contents from ```/home/ubuntu/.ssh/authorized_keys``` to grader's newly created public key storage file
        8. Restart SSH: ```sudo service ssh restart```
        9. SSH into VM using Amazon's default private key and command ```ssh -i [local location of private key] grader@35.175.151.242```
3. Disable remote login of root
    * ```sudo vi /etc/ssh/sshd_config```
    * Find and change ```PermitRootLogin without-password``` to ```PermitRootLogin no```
    * Restart SSH: ```sudo service ssh restart```
4. Change default SSH port to 2200
    * ```sudo vi /etc/ssh/sshd_config```
    * Find and change Port 22 to Port 2200
    * Restart SSH: ```sudo service ssh restart```
5. Setup universal firewall rules
    * Allow incoming tcp packets on port 2200: ```sudo ufw allow 2200/tcp```
    * Allow incoming tcp packets on port 80: ```sudo ufw allow 80/tcp```
    * Allow incoming udp packets on port 123: ```sudo ufw allow 123/udp```
    * Turn UFW on: ```sudo ufw enable```
    * Change Amazon Lightsail Instance Networking tab to reflect firewall configuration (Allow prots 80/TCP, 123/UDP, 2200/TCP and deny default SSH port 22)
    * Now SSH into VM using new port: ```ssh -i [local location of private key] -p 2200 grader@35.175.151.242```
6. PostgreSQL Config
    * Configure timezone: ```sudo timedatectl set-timezone UTC```
    * Install PostgreSQL: ```sudo apt-get install postgresql```
    * Check that no remote connections are allowd (should be default): ```sudo vi /etc/postgresql/9.5/main/pg_hba.conf```
    * Switch to *postgres* User: ```sudo su postgres```
    * Run psql shell: ```psql```
    * Create new database and user:
        ```sql
        CREATE USER catalog WITH PASSWORD 'meow';
        ALTER USER catalog CREATEDB;
        CREATE DATABASE catalog WITH OWNER catalog;
        \c catalog
        REVOKE ALL ON SCHEMA public FROM public;
        GRANT ALL ON SCHEMA public TO catalog;
        \q
        exit
        ```
7. Application Setup and Code Changes:
    * Install Git: ```sudo apt-get install git```
    * Create directory for FlaskApp: ```cd /var/www``` ```sudo mkdir FlaskApp```
    * Create directory for application files: ```cd FlaskApp``` ```git clone https://github.com/pika-hu/udacityfullstack-itemcatalog.git```
    * Rename directory to FlaskApp: ```mv udacityfullstack-itemcatalog/ FlaskApp/```
    * Among all python files, change ```engine = create_engine('sqlite:///aspirevolunteertracker.db')``` to ```engine = create_engine('postgresql://catalog:meow@localhost/catalog')```
    * Run ```database_setup.py``` and ```initialevents.py``` to setup database
    * Change ```project.py``` to ```__init__.py``` using ```mv``` command
    * In ```__init__.py``` update the path of ```client_secrets.json``` to the full path of directory (2 instances, at beginning and in gconnect() method)
    * In ```__init__.py``` change start of import statement ```from database_setup...``` to ```from .database_setup``` to allow for import of classes
    * Install Pip3: ```sudo apt-get install python3-pip```
    * Install and start VirutalEnv: ```sudo pip3 install virutalenv``` ```source venv3/bin/activate```
    * Install required packages: ```sudo pip3 install Flask httplib2 requests oauth2client sqlalchemy psycopg2 sqlalchemy_utils```
    * Run ```__init__.py``` to test local run of application
    * ```deactivate``` to exit venv
8. Deploy Flask App
    * Install Apache and Mod_wsgi: ```sudo apt-get install apache2``` ```sudo apt-get install libapache2-mod-wsgi python-dev```
    * Disable NGINX service (default installed on amazon instance and using HTTP socket): ```sudo service nginx stop```
    * Enable wsgi mod: ```sudo a2enmod wsgi```
    * Start Apache Service: ```sudo service apache2 start```
    * Configure Virtual Host by ```sudo vi /etc/apache2/sites-available/FlaskApp.conf``` and adding:
        ```
        <VirtualHost *:80>
                ServerName 35.175.151.242
                ServerAlias 35.175.151.242.xip.io
                ServerAdmin admin@35.175.151.242
                WSGIScriptAlias / /var/www/FlaskApp/flaskapp.wsgi
                <Directory /var/www/FlaskApp/FlaskApp/>
                        Order allow,deny
                        Allow from all
                </Directory>
                Alias /static /var/www/FlaskApp/FlaskApp/static
                <Directory /var/www/FlaskApp/FlaskApp/static/>
                        Order allow,deny
                        Allow from all
                </Directory>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
        </VirtualHost>
        ```
    * Enable the host ```sudo a2ensite FlaskApp```
    * Disable default host ```sudo a2dissite 000-default```
    * Create wsgi file by ```sudo vi /var/www/FlaskApp/flaskapp.wsgi``` and adding:
        ```
        #!/usr/bin/python3
        import sys
        import logging
        logging.basicConfig(stream=sys.stderr)
        sys.path.insert(0,"/var/www/FlaskApp/")

        from FlaskApp import app as application
        application.secret_key = 'super_secret_key'
        ```
    * Restart Apache service: ```sudo service apache2 restart```

## File Structure
```
── FlaskApp
   ├── FlaskApp
   │   ├── client_secrets.json
   │   ├── database_setup.py
   │   ├── initialevents.py
   │   ├── __init__.py
   │   ├── __pycache__
   │   │   └── database_setup.cpython-35.pyc
   │   ├── README.md
   │   ├── static
   │   │   └── styles.css
   │   ├── templates
   │   │   ├── deleteEvent.html
   │   │   ├── deleteVolunteer.html
   │   │   ├── editEvent.html
   │   │   ├── editVolunteer.html
   │   │   ├── events.html
   │   │   ├── header.html
   │   │   ├── login.html
   │   │   ├── main.html
   │   │   ├── newEvent.html
   │   │   ├── newVolunteer.html
   │   │   └── volunteers.html
   │   └── venv3
   │       ├── bin
   │       ├── include
   │       └── lib
   └── flaskapp.wsgi
```

## Third Party Resources
* [How to Secure PostgreSQL on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)
* [How To Deploy a Flask Application on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
* [Xip.io](http://xip.io/)
