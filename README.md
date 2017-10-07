# Linux Server Configuration

In this project, I took a baseline Linux server on a virtual machine and configured it as a secure server to host my Plant Catalog web application.

The IP address of my server is: 34.214.109.173

SSH port: 2200

The complete URL to my hosted web application is:
[http://ec2-34-214-109-173.us-west-2.compute.amazonaws.com](http://ec2-34-214-109-173.us-west-2.compute.amazonaws.com)

### How I Configured the Server

1. Created a new Ubuntu Linux server instance on [Amazon Lightsail](https://lightsail.aws.amazon.com) and followed their instructions to SSH into the server.

2. Upgraded all currently installed packages.

  * `sudo get-apt update`
  * `sudo get-apt upgrade`

3. Changed the SSH port from 22 to 2200.

  * `sudo nano /etc/ssh/sshd_config` to add a new line for "Port 2200"
  * Configured the external firewall under the Networking tab on the Lightsail webpage by adding a custom rule to allow tcp traffic on Port 2200
  * Once I configure the UFW (See step 4) and confirm that I can login on Port 2200, then I can disable Port 22

4. Configured the Uncomplicated Firewall to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

  * `sudo ufw default deny incoming`
  * `sudo ufw default allow outgoing`
  * `sudo ufw allow ssh`
  * `sudo ufw allow 2200/tcp`
  * `sudo ufw allow www`
  * `sudo ufw allow ntp`
  * `sudo ufw enable`
  * `sudo service sshd restart`  

  * Tested that can login to both Port 22 and Port 2200
  * Deny access to Port 22 by executing the following:
    * `sudo ufw deny 22`
    * Edit `/etc/ssh/sshd_config` to remove "Port 22"
    * Remove Port 22 from Amazon's Lightsail webpage using their Networking GUI
    * `sudo service sshd restart`

5. Created a new user account named grader.

  * `sudo apt-get install finger`
  * `sudo adduser grader`

6. Gave the grader permission to sudo.

  * Created a file `/etc/sudoers.d/grader` with contents `grader ALL=(ALL) NOPASSWD:ALL`

7. Created an SSH key pair for the grader using the ssh-keygen tool.

  * On my local Git Bash, ran `ssh-keygen` and created key file in `~/.ssh/linuxProject` with a passphrase
  * On server in `/home/grader`, ran `sudo mkdir .ssh`
  * `touch .ssh/authorized_keys` and copied contents of `~/.ssh/linuxProject.pub` into `authorized_keys`
  * `chmod 700 .ssh`
  * `chmod 644 .ssh\authorized_keys`
  * Since this .ssh directory and key file were created as root, I used the `chown` and `chgrp` commands to change owner and group of both to `grader`
  * I did not have to disable password login, since `PasswordAuthentication` was already set to "no" in `/etc/ssh/sshd_config`

8. Disabled remote root login

  * In `/etc/ssh/sshd_config` set `PermitRootLogin` to "no"
  * `sudo service sshd restart` to execute these changes

9. Configured the local timezone to UTC.

  * `sudo timedatectl set-timezone UTC`
  * `timedatectl status` confirms that local time is UTC

10. Installed and configured Apache to serve a Python mod_wsgi application.

  * `sudo apt-get install apache2`
  * Visited `http://34.214.109.173` in browser and checked for the Ubuntu default page
  * `sudo apt-get install libapache2-mod-wsgi`
  * Edited `/etc/apache2/sites-enabled/000-default.conf` to include the line `WSGIScriptAlias / /var/www/html/myapp.wsgi`
  * Created `myapp.wsgi` to print a "Hello World" message
  * Restarted apache with `sudo apache2ctl restart` and tested for message by visiting my website in browser

11. Installed and configured PostgreSQL

  * `sudo apt-get install postgresql`
  * `sudo -u postgres createuser --interactive` to create the "catalog" user by setting the "role" to "catalog" and all the rest of the prompts for superuser, database creation, and role creation to "n"
  * `sudo -u postgres createdb -O catalog catalog` to create the catalog database and set user "catalog" as its owner
  * Checked `/etc/postgresql/9.5/main/pg_hba.conf` to confirm that all connections are local and no remote hosts are allowed
  * `sudo -u postgres psql` to login into pqsl
  * Within psql, set a password for user "catalog" using `alter user catalog password 'insert-actual-password-here';`
  * Within psql, run `\du` command to show users and confirm that user "catalog" exists with no special privileges
  * Within psql, run `\l` command to list the databases. It shows a database "catalog" owned by user "catalog".

12. Installed git

  * `sudo apt-get install git`

13. Cloned and set up my Plant Catalog project on this server

  * Run `sudo mkdir` to create directory `/var/www/catalog`
  * Go to this new directory and clone the Plant Catalog project here using `git clone https://github.com/chspanos/catalog`. This installed all my Python code and assets in `/var/www/catalog/catalog`
  * Create `/var/www/catalog/catalog/__init__.py` which should be empty. This is needed to configure Python packages.
  * `sudo apt-get install python-pip`
  * `sudo -H pip install flask`
  * `sudo -H pip install SQLAlchemy`
  * `sudo apt-get install python-psycopg2`
  * `sudo -H pip install oauth2client`
  * `sudo -H pip install requests`
  * `sudo -H pip install bleach`

14. Setup the application to create and build the plant database

  * In `database_setup.py`, `lotsofplants.py`, and `application.py`, changed the line that reads `engine = create_engine('sqlite:///plantcatalog.db')` to`engine = create_engine('postgresql://catalog:password@localhost/catalog')`, inserting the actual password for user "catalog" after the ":"
  * Ran `python database_setup.py` to create the database
  * Logged in to the database using `sudo -u postgres psql` and ran `\c catalog` to connect to the "catalog" database and `\dt` to display the tables.
  * Ran `python lotsofplants.py` to populate the database with initial data
  * Again checked the database to see that the data had been loaded

15. Configured the application in Flask

  * Created a `catalog.wsgi` file in `/var/www/catalog` with the following contents:

  ```
  #!/usr/bin/python
  import sys
  import logging
  logging.basicConfig(stream=sys.stderr)
  sys.path.insert(0, "/var/www/catalog/")

  from catalog.project import app as application
  application.secret_key = 'insert-secret-key-here'
  ```
  * Modified `/etc/apache2/sites-enabled/000-default.conf`to contain the following:

  ```
  ServerName 34.214.109.173
  ServerAdmin webmaster@localhost
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
  ```
  * To avoid naming conflicts between my Python program and the running web application, I changed the name of my Python project file in `/var/www/catalog/catalog` from `application.py` to `project.py`
  * Restarted the server with `sudo apache2ctl restart`
  * Visiting my site in a browser confirmed that my application was running on the server

16. Configured the application to correctly handle 3rd party Oauth

  * Visited [https://console.developers.google.com/apis](https://console.developers.google.com/apis) and updated the credentials for this application. Specifically, added my website IP address to the allowed javascript origins and updated the callback URLs for URI redirect
  * Downloaded the new client_secrets.json file from Google and copied it to `/var/www/catalog/catalog/client_secrets.json`
  * Changed the relative path to `client_secrets.json` to an absolute path in `project.py`. Note this change had to be made in two places in the program.
  * Restarted the server with `sudo apache2ctl restart` and tested the application.

17. Made sure the .git directory is not publically accessible

  * `sudo chmod 700 .git`

### Resources Used

This project was completed as part of Udacity's Full-Stack Web Developer nanodegree.

The following is a list of some of the resources that I used to complete this project:
* Udacity class **Configuring Linux Web Servers**, part of Udacity's Full-Stack Web Developer nanodegree
* Udacity Forums, [FSND: Linux-based Server Configuration](https://discussions.udacity.com/c/nd004-p7-linux-based-server-configuration)
* Digital Ocean tutorial "How to Deploy a Flask Application on an Ubuntu VPS" located [here](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps)
* Flask mod_wsgi (Apache) documentation [http://flask.pocoo.org/docs/0.10/deploying/mod_wsgi/](http://flask.pocoo.org/docs/0.10/deploying/mod_wsgi/)
* Digital Ocean tutorial "How to Install and Use PostgreSQL on Ubuntu" located [here](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04)
* Engine configuration for SQLAlchemy [http://docs.sqlalchemy.org/en/latest/core/engines.html](http://docs.sqlalchemy.org/en/latest/core/engines.html)
* [Google](https://google.com) and [StackOverflow](https://stackoverflow.com/) for help in deciphering some of the error messages I received in `/var/log/apache2/error.log`
