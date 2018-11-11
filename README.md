# Udacity Full Stack Web Developer Nanodegree Project 7: Linux_Server_Configuration
Set-up information for Udacity Full Stack Nanodegree project on how to configure a Linux Server.

## Project Description

> Taking a baseline installation of a Linux distribution on a virtual machine and prepare it to host your web applications, to include installing updates, securing it from a number of attack vectors and installing/configuring web and database servers.

The application deployed here is the **Item Catalog - Frenchy Fabric**, previously developed for [Item-Catalog Project](https://github.com/swtsprt247/FrenchyFabric).

## Server/App Info

IP address: 54.162.62.114
SSH port: 2200.

Application URL: [http://ec2-54-162-62-114.compute-1.amazonaws.com](http://ec2-54-162-62-114.compute-1.amazonaws.com).

Username and password for Udacity reviewer: `grader`,


### 1 - Launching an AWS Lightsail Instance and connect to it via SSH

1. Launching an AWS Lightsail instance
2. The instance's security group provides a SSH port 22 by default
3. The public IP is 54.162.62.114
4. Download the private key `LightsailDefaultPrivateKey.pem` from AWS

### 2 - User, SSH and Security Configurations

1. Log into the remote VM as *root* user (`ubuntu`) through ssh: `$ ssh grader@54.162.62.114 -p 2200 -i /LightsailDefaultPrivateKey.pem`.
2. Create a new user *grader*:  `$ sudo adduser grader`.
3. Grant udacity the permission to sudo, by adding a new file under the suoders directory: `$ sudo nano /etc/sudoers.d/grader`. In the file put in: `grader ALL=(ALL:ALL) ALL`, then save and quit.
4. Generate a new key pair by entering the following command at the terminal of your *local machine*.
    1. `$ ssh-keygen` with `grader_key`
    2. Print the public key `$ cat ~/.ssh/grader_key.pub`.
    3. Select the public key and copy it. 
    4. deploy public key on developement enviroment

	On you virtual machine:
	```
	$ su - grader
	$ mkdir .ssh
	$ touch .ssh/authorized_keys
	$ nano .ssh/authorized_keys
	```
	Copy the public key generated on your local machine to this file and save
	```
	$ chmod 700 .ssh
	$ chmod 644 .ssh/authorized_keys
	```
    5. Change the owner from `ubuntu` to `grader`: `$ sudo chown -R grader:grader /home/grader/.ssh`
  
3. reload SSH using `sudo service ssh restart`
4. now you can use ssh to login with the new user you created

	`ssh -i [privateKeyFilename] grader@54.237.147.20`

### 3. Update all currently installed packages

	sudo apt-get update
	sudo apt-get dist-upgrade
	sudo apt-get upgrade
	
- Configure cron scripts to automatically update packages (Exceeds Specifications)

1. Install *unattended-upgrades*: `$ sudo apt-get install unattended-upgrades`.
2. Enable it by: `$ sudo dpkg-reconfigure --priority=low unattended-upgrades`.

Source: [Ubuntu Server Guide](https://help.ubuntu.com/12.04/serverguide/automatic-updates.html).
  
  
### 3 - Configure the local timezone to UTC
  * Change the timezone to UTC using following command: `$ sudo timedatectl set-timezone UTC`.
  * You can also open time configuration dialog and set it to UTC with: `$ sudo dpkg-reconfigure tzdata`.
  * Install ntp daemon ntpd for a better synchronization of the server's time over the network connection:
  
  ```
   $ sudo apt-get install ntp
  ```
 Source: [UbuntuTime](https://help.ubuntu.com/community/UbuntuTime)
 
 
 ### 4 -  Change the SSH port from 22 to 2200
. Enforce key-based authentication, change SSH port to `2200` and disable remote login of *root* user:
   1. `$ sudo nano /etc/ssh/sshd_config`  
   2. Change `PasswordAuthentication` to `no`.
   3. Change `Port` to `2200`.
   4. Change `PermitRootLogin` to `no`
   5. `$ sudo service ssh restart`.
   6. In AWS Lightsail Security Group,  add `2200` as the inbound custom TCP Rule port.
 
  
### 5 - Configure the Uncomplicated Firewall (UFW)

Project requirements need the server to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

1. `$ sudo ufw default deny incoming`.
2. `$ sudo ufw default allow outgoing`.
3. `$ sudo ufw allow 2200/tcp`.
4. `$ sudo ufw allow 80/tcp`.
5. `$ sudo ufw allow 123/udp`.
6. `$ sudo ufw enable`.
7. Add 3 rules above as Security Group inbound rules of AWS Lightsail instance

### 6 - Install Apache, mod_wsgi and Git

1. `$ sudo apt-get install apache2`.
2. Install *mod_wsgi* with the following command: `$ sudo apt-get install libapache2-mod-wsgi python-dev`
3. Enable *mod_wsgi*: `$ sudo a2enmod wsgi`
4. `$ sudo service apache2 start`
5. `$ sudo apt-get install git`


### 7 - Install and configure PostgreSQL
  - `sudo apt-get install libpq-dev python-dev`
  - `sudo apt-get install postgresql postgresql-contrib`
  - `sudo su - postgres`
  - `psql`
  - `CREATE USER catalog WITH PASSWORD 'password';`
  - `ALTER USER catalog CREATEDB;`
  - `CREATE DATABASE catalog WITH OWNER catalog;`
  - `\c catalog`
  - `REVOKE ALL ON SCHEMA public FROM public;`
  - `GRANT ALL ON SCHEMA public TO catalog;`
  - `\q`
  - `exit`
  - Change create engine line in your `__init__.py` and `database_setup.py` to: 
  `engine = create_engine('postgresql://catalog:password@localhost/catalog')`
  - `python /var/www/catalog/catalog/database_setup.py`
  
  ### 8 - Configure Apache to serve a Python mod_wsgi application

1. Clone the item-catalog (Frenchy Fabric) app from Github
   ```
   $ cd /var/www
   $ sudo mkdir catalog
   $ sudo chown -R grader:grader catalog
   $ cd catalog
   $ git clone https://github.com/swtsprt247/Frenchy-Fabric-Server.git catalog
  
  2. Install pip , virtualenv (in /var/www/catalog)
   ```
   $ sudo apt-get install python-pip
   $ sudo pip install virtualenv
   $ sudo virtualenv venv
   $ source venv/bin/activate
   $ sudo chmod -R 777 venv
   ```
3. Install Flask and other dependencies:
   ```
   $ sudo pip install -r catalog/requirements.txt
   ```
4. Install Python's PostgreSQL adapter *psycopg2*:
   ```
   $ sudo apt-get install python-psycopg2
   ```
5. Configure and Enable a New Virtual Host
   ```
   $ sudo nano /etc/apache2/sites-available/catalog.conf
   ```
   Add the following content:
```
<VirtualHost *:80>
		ServerName 54.162.62.114
		ServerAdmin admin@54.162.62.114
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
6. Create and configure the .wsgi File
   ```
   $ cd /var/www/catalog/
   $ sudo nano catalog.wsgi
   ```
   Add the following content:
  ```
  #!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/catalog/")

from catalog import app as application
application.secret_key = 'Add your secret key'
```
* Directory structure should be :
```
|--------catalog
|----------------catalog
|-----------------------static
|-----------------------templates
|-----------------------venv
|-----------------------__init__.py
|----------------catalog.wsgi

```

### 9- Run the application:
* Create the datbase schema:
    `python database_setup.py`
    `python fabricfabric.py`
* Restart Apache : `sudo service apache2 restart`
If an internal error shows up when you try to access the app, open Apache error log as a reference for debugging:
   ```
   $ sudo tail -20 /var/log/apache2/error.log
   ```
   
### 10 - Install system monitor tools (Exceeds Specifications)

1. `$ sudo apt-get update`.
2. `$ sudo apt-get install glances`.
3. To start this system monitor program: `$ glances`.

Source: [Askubuntu](http://askubuntu.com/questions/293426/system-monitoring-tools-for-ubuntu).


