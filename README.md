# Deploy of a flask app with postgres in aws lightsail
This project is a requirement of Udacity fullstack nano degree. 

# Step by Step Guide

## Create an aws lightsail instance
- Enter https://lightsail.aws.amazon.com/ls/webapp/home/instances
- Choose: [Create Instance] -> [Linux] -> [Ubuntu 16.04] -> [Create Instance]
- Wait "Pending" status change

## Create a key pair (in your local machine)
I'm using gitbash in a windows machine. In gitbash prompt:
```sh
$ ssh-keygen
```
Choose file to save the key: /c/Users/<YOUR_WINDOWS_USERNAME>/.ssh/my_key

### Create a new user (in aws console)
- In Networking (aws panel) set another rule in firewall:
-- Application = Custom
-- Protocol = TCP
-- Port = 2200

Thanks to CharInt: 
https://askubuntu.com/questions/1019891/connecting-to-amazon-lightsail-ubuntu-server-using-different-ssh-port)
- Login in aws ssh console and type the following:
```sh
$ sudo adduser grader
$ sudo touch /etc/sudoers.d/grader
$ sudo nano /etc/sudoers.d/grader
```
- Paste in grader file the following:
*grader ALL=(ALL:ALL) ALL*
- CTRL+O (save), ENTER (confirm), CTRL+X (exit nano)
User grader have sudo privilegies now! Let's authorize grader to do remote login. In aws ssh:
```sh
$ su - grader
$ mkdir .ssh
$ sudo touch .ssh/authorized_keys
$ sudo nano .ssh/authorized_keys
```
- Copy the public key generated on your local machine to this file
- CTRL+O (save), ENTER (confirm), CTRL+X (exit nano)
- Again, in aws console:
```sh
$ sudo chown -R  grader.grader /home/grader/.ssh
$ sudo chmod  700 /home/grader/.ssh
$ sudo chmod 600 /home/grader/.ssh/authorized_keys
$ ls -als .ssh/
```
Must print:
```
4 drwx------ 2 grader grader 
4 drwxr-xr-x 4 grader grader 
4 -rw------- 1 grader grader 
```
Now, let's restart ssh and change its port to 2200:
```sh
$ sudo service ssh restart
$ sudo nano /etc/ssh/sshd_config
```
- Change *Port* to *2200*
- CTRL+O (save), ENTER (confirm), CTRL+X (exit nano)
```sh
$ sudo service sshd restart
$ sudo service ssh restart
```
 You can login with grader from your windows machine using gitbash now:
 ```sh
$ ssh grader@54.160.19.10 -p 2200 -i ~/.ssh/my_key
```
Here 54.160.19.10 must be change to your Public IP. We change the usual ssh port to 2200.

## UPDATE and UPGRADE
Logged as grader:
 ```sh
$ sudo apt-get update
$ sudo apt-get upgrade
```

## Disable root login
Logged as grader:
 ```sh
$ sudo vim /etc/ssh/sshd_config
```
- Change *PermitRootLogin* to *no*
- CTRL+O (save), ENTER (confirm), CTRL+X (exit nano)

## Configure FIREWALL
Logged as grader:
 ```sh
$ sudo ufw allow 2200/tcp
$ sudo ufw allow 80/tcp
$ sudo ufw allow 123/udp
$ sudo ufw enable
```

## Configure UTC TIME_ZONE
Logged as grader:
 ```sh
$ sudo dpkg-reconfigure tzdata
```

## Install APACHE 2
Logged as grader:
 ```sh
$ sudo apt-get install apache2
$ sudo apt-get install libapache2-mod-wsgi-py3 python3-dev
```
Enter in your navigator with *YOUR_PUBLIC_IP_ADDRESS* (Mine is 54.160.19.10) to test if apache works fine.

## Install Python 3 requirements
Logged as grader:
 ```sh
$ sudo apt-get install python3-pip
$ sudo python3 -m pip install --upgrade pip
$ pip3 install <PYTHON PACKAGES>
```
The necessary python packages for this project are flask, sqlalchemy and psycopg2. The last one have some problems because dependencies. To solve this:
 ```sh
$ sudo apt-get install libpq-dev python3-dev 
$ pip3 install psycopg2
```
Thanks Muhammad Taqi: 
https://stackoverflow.com/questions/28253681/you-need-to-install-postgresql-server-dev-x-y-for-building-a-server-side-extensi)

## Install and Config POSTGRESQL
Logged as grader:
```sh
$ sudo apt-get install postgresql
$ sudo su - postgres
(postgres)$ psql
postgres=# CREATE DATABASE grader;
postgres=# CREATE USER grader;
postgres=# ALTER ROLE grader WITH PASSWORD 'password';
postgres=# GRANT ALL PRIVILEGES ON DATABASE grader TO grader;
postgres=# \q
(postgres)$ logout
```
Now, your have a *grader* database and a *grader* user tiwh all privileges on this database.

## Clone your Flask App
Logged as grader:
```sh
$ sudo apt-get install git
$ cd /var/www
$ sudo mkdir FlaskApp
$ cd FlaskApp
$ git clone https://github.com/leandrocl2005/restaurant_python3_udacity.git
$ sudo mv ./restaurant_python3_udacity ./FlaskApp
$ cd FlaskApp
$ sudo nano config.py
```
Change DATABASE_URL to 'postgresql://grader:password@localhost/grader'
```sh
$ sudo nano app.py
```
Remove *threaded=False* in line 138.

## Create your Apache configuration file
Logged as grader:
```sh
$ sudo nano /etc/apache2/sites-available/FlaskApp.conf
```
Copy and past the following configurations:
```sh
<VirtualHost *:80>
        ServerName 54.160.19.10
        ServerAdmin leandrocl2005@yahoo.com
        ServerAlias 54.160.19.10.xip.io
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
## Create wsgi configuration file
Logged as grader:
```sh
$ cd /var/www/FlaskApp
$ sudo nano flaskapp.wsgi 
```
Copy and past the following code:
```sh
#!/usr/bin/python3
import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/FlaskApp/FlaskApp")

from app import app as application
application.secret_key = 'super secret key'
```
Restart and reload Apache:
```sh
$ sudo service apache2 restart
$ sudo service apache2 reload
```

## Test your flask app
Enter in your navigator with *YOUR_PUBLIC_IP_ADDRESS.xip.io* (Mine is 54.160.19.10.xip.io) to test if apache works fine. If something works bad, logged as user you can check the apache error.log file:
```sh
sudo tail -f /var/log/apache2/error.log 
```

# Thanks
- To kongling893 for his excelent step by step guide: https://github.com/kongling893/Linux-Server-Configuration-UDACITY/blob/master/README.md
