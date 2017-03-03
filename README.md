# Project: Linux Server Configuration

A baseline installation of a Linux distribution on a virtual machine to host a Flask web application. This includes the installation of updates, securing it from a number of attack vectors and installing/configuring web and database servers.


## Server details
- SSH port: `2200`
- IP address: `52.32.134.3`
- URL: `http://ec2-52-32-134-3.us-west-2.compute.amazonaws.com/`
-> It is not working now

## Configuration changes
### Create a new user named grader
`sudo adduser grader`

### Give the grader the permission to sudo
  - -G = To add a supplementary groups.
  - -a = To add anyone of the group to a secondary group.
`sudo usermod -aG sudo grader`

### Update all currently installed packages
```
apt-get update
apt-get upgrade
```

### Set up SSH keys for user *grader*
```
mkdir /home/grader/.ssh
chown grader:grader /home/grader/.ssh
chmod 700 /home/grader/.ssh
cp /root/.ssh/authorized_keys /home/grader/.ssh/
chown grader:grader /home/grader/.ssh/authorized_keys
chmod 644 /home/grader/.ssh/authorized_keys
```

- Login using `ssh -i ~/.ssh/udacity_key.rsa grader@52.32.134.3` command

### Change the SSH port from 22 to 2200
`sudo nano /etc/ssh/sshd_config`

- Login using `ssh -i ~/.ssh/udacity_key.rsa grader@52.32.134.3 -p 2200` command

### Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
  1. Block all incoming connections :
  `sudo ufw default deny incoming`
  2. Allow outgoing connections :
  `sudo ufw default allow outgoing`
  3. Allow incoming connections for SSH (port 2200) :
  `sudo ufw allow 2200/tcp` or `sudo ufw allow ssh`
  4. Allow incoming connections for HTTP (port 80) :
  `sudo ufw allow www`
  5. Allow incoming connections for NTP (port 123) :
  `sudo ufw ntp`
  6. Enable the firewall :
  `sudo ufw enable`

### Configure the local timezone to UTC
`sudo timedatectl set-timezone UTC`

### Install Apache to serve a Python mod_wsgi application
```
sudo apt-get install apache2
sudo apt-get install libapache2-mod-wsgi
sudo a2enmod wsgi
```

### Install modules & packages
```
source venv/bin/activate
sudo pip install httplib2
sudo pip install requests
sudo pip install oauth2client
sudo pip install sqlalchemy
sudo apt-get install python-psycopg2
```

### Install git and clone my Item Catalog project
  - Install git :
  `sudo apt-get install git`
  - Clone my Item Catalog project :
  `sudo git clone https://github.com/eunbigo91/project5-item-catalog.git /var/www/catalog/catalog`
  - Prevent .git directory from being publicly accessed :
  ```
  sudo nano /var/www/catalog/catalog/.git/htaccess

  <Directory .git>
    order allow,deny
    deny from all
  </Directory>
  ```

### Install and configure PostgreSQL
  - Install PostgreSQL :
  `sudo apt-get install postgresql`
  - Check remote connection is not allowed :
  `sudo cat /etc/postgresql/9.3/main/pg_hba.conf`
  - Create a new user named 'catalog' :
  `sudo -u postgres createuser -p catalog`
  - Give a permission to my catalog application database :
  ```
  sudo -u postgres psql
  
  CREATE USER catalog WITH PASSWORD 'passwd';
  
  ALTER USER catalog CREATEDB;
  
  CREATE DATABASE catalog WITH OWNER catalog;
  
  \c catalog
  
  REVOKE ALL ON SCHEMA public FROM public;
  
  GRANT ALL ON SCHEMA public TO catalog;
  
  \q
  ```
  - Create postgreSQl database schema :
  `python database_setup.py`
  - Open the database setup file and change the line with 'engine' :
  ```
  sudo nano database_setup.py  
  
  engine = create_engine('postgresql://catalog:passwd@localhost/catalog')
  ```
  - Open applicaion.py and change the same line :
  `sudo nano application.py`
  - Rename applicaion.py :
  `mv application.py __init__.py`

### Setup to deploy a Flask Application on Ubuntu VPS [Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
  1. Enable mod_wsgi : `$ sudo a2enmod wsgi`
  2. Install virtualenv and Flask. If pip is not installed, install it on Ubuntu through apt-get : `sudo apt-get install python-pip`
  3. If virtualenv is not installed, use pip to install : `sudo pip install virtualenv`
  4. Set virtual environment to name 'venv' : `sudo virtualenv venv`
  5. Enable all permissions : `sudo chmod -R 777 venv`
  6. Install Flask in that environment by activating the virtual environment : `source venv/bin/activate`
  7. Install Flask inside the virtual environment : `sudo pip install Flask`
  8. Run the app : `python __init__.py`

  To deactivate the environment, give the following command :
  `deactivate`

  9. Create a virtual host configure file :
  `sudo nano /etc/apache2/site-available/catalog.conf`
  ```
  <VirtualHost *:80>
        ServerName 52.32.134.3
        ServerAlias ec2-52-32-134-3.us-west-2.compute.amazonaws.com
        ServerAdmin admin@52.32.134.3
        WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/$
        WSGIProcessGroup catalog
        WSGIScriptAlias / /var/www/catalog/catalog.wsgi

        <Directory /var/www/catalog/catalog/>
            Require all granted
        </Directory>

        Alias /static /var/www/catalog/catalog/static
        <Directory /var/www/catalog/catalog/static/>
            Require all granted
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
  </VirtualHost>
  ```
  10. Enable the virtual host :
  `sudo a2dissite 000-default.conf
  sudo a2ensite catalog`
  11. Create and configure 'catalog.wsgi' :
  `sudo nano /var/www/catalog/catalog.wsgi`
  ```
  #!/usr/bin/python
  import sys
  import logging
  logging.basicConfig(stream=sys.stderr)
  sys.path.insert(0, '/var/www/catalog/')

  from catalog import app as application
  application.secret_key = 'IS_THIS_REALLY_SECRET'
  ```
  12. Restart Apache:
  `sudo service apache2 restart`

### Update the OAuth client secrets file (Google & Facebook)
Change the path 'client_secrets.json' and 'fb_client_secrets.json' in __init__.py to '/var/www/catalog/catalog/client_secrets.json' and '/var/www/catalog/catalog/fb_client_secrets.json'.
#### Google
  1. Go to [Google APIs](https://console.developers.google.com/apis)
  2. API Manager > Credentials
  3. Add url and IP address to Authorized JavaScript origins.
  4. Add 'url+oauth2callback' to Authorized redirect URIs

#### Facebook
  1. Go to [Facebook for developers](https://developers.facebook.com/apps/)
  2. Click your app, and go to settings and fill in your IP address

### Run application
1. Restart Apache :
`sudo service apache2 restart`
2. Open a brower and put ip-address [52.32.134.3](http://52.32.134.3/) or [url](http://ec2-52-32-134-3.us-west-2.compute.amazonaws.com/).


## Third party library
[PostgreSQL](https://www.postgresql.org/)  
[Flask](http://flask.pocoo.org/)  
[Google OAuth 2.0](https://console.developers.google.com/apis)  
[Facebook OAuth](https://developers.facebook.com/apps/)  


