# Notes for the reviewer

Connect to this instance with the following command:

    $ ssh grader@52.39.235.152 -p 2200 -i <private key provided in reviewer notes>

And view the final instance at:

http://ec2-52-39-235-152.us-west-2.compute.amazonaws.com/

# Configure the base system

## Update all the existing packages

Login to the remote machine as the standard user:

    (local) root$ ssh -i ~/.ssh/udacity_key.rsa root@52.39.235.152

    (remote) root$ apt-get update
    (remote) root$ apt-get upgrade
    (remote) root$ apt-get autoremove

## Configure a new 'grader' user

    (remote) root$ adduser grader

### Add the 'grader' user to sudoers

    (remote) root$ nano /etc/sudoers.d/grader

    -> /etc/sudoers.d/grader contents:

    grader ALL=(ALL) NOPASSWD:ALL

### Generate an SSH key for the 'grader' user

On my local machine:

    (local) root$ ssh-keygen -t rsa -b 2048

    -> Key stored in udacity_key_grader.rsa

And copy the key contents into my local clipboard:

    (local) root$ pbcopy < ~/.ssh/udacity_key_grader.rsa.pub

### Store the new SSH key for the 'grader' user

Back on the remote machine:

    (remote) root$ cd /home/grader/
    (remote) root$ mkdir .ssh
    (remote) root$ nano .ssh/authorized_keys

And paste the key from my local clipboard into the authorized_keys file.

### Set the proper owner and group for the 'grader' user's .ssh contents

    (remote) root$ chown grader .ssh/
    (remote) root$ chown grader .ssh/authorized_keys
    (remote) root$ chgrp grader .ssh/
    (remote) root$ chgrp grader .ssh/authorized_keys

### Login as the 'grader' user to set the proper permissions for the .ssh folder and contents

    (local) root$ ssh -i ~/.ssh/udacity_key_grader.rsa grader@52.39.235.152

    (remote) grader$ chmod 700 ~/.ssh
    (remote) grader$ chmod 644 ~/.ssh/authorized_keys

## Configure the SSH rules to secure access to the computer

    (remote) grader$ sudo nano /etc/ssh/sshd_config

    -> Ensure the following are set:

    Port 2200
    PermitRootLogin no
    PasswordAuthentication no

    (remote) grader$ sudo service ssh restart

## Configure firewall rules

    (remote) grader$ sudo ufw default deny incoming
    (remote) grader$ sudo ufw default allow outgoing
    (remote) grader$ sudo ufw allow 2200/tcp
    (remote) grader$ sudo ufw allow www
    (remote) grader$ sudo ufw allow ntp
    (remote) grader$ sudo ufw enable

## Set the system timezone to UTC if not set to UTC already

    sudo dpkg-reconfigure tzdata

    -> Resource used: http://askubuntu.com/questions/3375/how-to-change-time-zone-settings-from-the-command-line

# Configure the Catalog application

## Install Apache

    (remote) grader$ sudo apt-get install apache2

## Install WSGI

    (remote) grader$ sudo apt-get install libapache2-mod-wsgi

## Install PostgreSQL

    -> Resource used: https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps
    -> Resource used: https://www.digitalocean.com/community/tutorials/how-to-use-roles-and-manage-grant-permissions-in-postgresql-on-a-vps--2

### Install the main PostgreSQL packages

    (remote) grader$ sudo apt-get install postgresql postgresql-contrib

### Configure PostgreSQL to ensure only local connections are allowed

    (remote) grader$ sudo nano /etc/postgresql/9.3/main/pg_hba.conf

    -> Ensure the configuration table matches the following

    local   all             postgres                                peer
    local   all             all                                     peer
    host    all             all             127.0.0.1/32            md5
    host    all             all             ::1/128                 md5

### Setup a new role for the catalog

    (remote) grader$ sudo su - postgres

    -> User is switched to the postgres user

    (remote) postgres$ psql

    (remote) postgres=# create role catalog with login;
    (remote) postgres=# alter role catalog with password 'catalog';
    (remote) postgres=# create database catalog with owner catalog;
    (remote) postgres=# \c catalog;

    (remote) catalog=# revoke all on schema public from public;
    (remote) catalog=# grant all on schema public to catalog;
    (remote) catalog=# \q

## Get the Catalog project

### Install Git

    (remote) grader$ sudo apt-get install git

### Clone the Catalog repo into the 'grader' user's home directory

    (remote) grader$ git clone https://github.com/hwyfour/fullstack-nanodegree-vm.git

### Symlink the Catalog subdirectory into a new folder in the www folder

    (remote) grader$ sudo mkdir /var/www/catalog
    (remote) grader$ sudo ln -s /home/grader/fullstack-nanodegree-vm/vagrant/catalog/ /var/www/catalog

### Install the dependencies for the Catalog project

    (remote) grader$ sudo apt-get install python-pip python-dev libpq-dev
    (remote) grader$ sudo pip install requests psycopg2 flask oauth2client sqlalchemy

## Reconfigure the Catalog project

### Change the Catalog database to use PostgreSQL instead of SQLite

In database_setup.py, application.py, and feedme.py, change:

    engine = create_engine('sqlite:///catalog.db')

To:

    engine = create_engine('postgresql://catalog:catalog@localhost/catalog')

### Change the relative link for the client_secret.json to absolute

In application.py, change:

    client_id = json.loads(open('client_secret.json', 'r').read())['web']['client_id']

To:

    client_id = json.loads(open('/var/www/catalog/catalog/client_secret.json', 'r').read())['web']['client_id']

### Replace the client_secret.json with a new one from Google to enable OAuth

This process is done on `https://console.developers.google.com/` and requires that all endpoint
URLs are changed to `http://ec2-52-39-235-152.us-west-2.compute.amazonaws.com/`.

### Rename the Catalog application.py file so we can import it as a package

    (remote) grader$ cd ~/fullstack-nanodegree-vm/vagrant/catalog/
    (remote) grader$ mv application.py __init__.py

## Create the tables for the Catalog project and fill them with some data

    (remote) grader$ cd ~/fullstack-nanodegree-vm/vagrant/catalog/
    (remote) grader$ python database_setup.py
    (remote) grader$ python feedme.py

## Create a WSGI for the Catalog project

    (remote) grader$ cd /var/www/catalog
    (remote) grader$ sudo nano catalog.wsgi

    -> /var/www/catalog/catalog.wsgi contents:

    #!/usr/bin/python
    import os
    import sys
    import logging

    logging.basicConfig(stream = sys.stderr)
    sys.path.insert(0, "/var/www/catalog/")

    from catalog import app as application
    application.secret_key = os.urandom(20)

## Configure WSGI

    -> Resource used: https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps

### Make a WSGI configuration for the Catalog project

    (remote) grader$ sudo nano /etc/apache2/sites-available/catalog.conf

    -> /etc/apache2/sites-available/catalog.conf contents:

    <VirtualHost *:80>
        ServerName 52.39.235.152
        ServerAdmin hwyfour@gmail.com
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

### Enable the Catalog configuration

    (remote) grader$ sudo a2ensite catalog

## Disable the default Apache website

    -> Resource used: http://stackoverflow.com/questions/23713061/virtualhost-always-returns-default-host-with-apache-on-ubuntu-14-04

    (remote) grader$ sudo a2dissite 000-default.conf

## Reload the Apache configuration

    (remote) grader$ sudo service apache2 reload

## View in your browser!

http://ec2-52-39-235-152.us-west-2.compute.amazonaws.com/
