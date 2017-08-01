# An FSND Linux server configuration project

My instance is located at [gideonservice.me](http://gideonservice.me/), with IP address 46.101.150.121.

The ssh key for grader is given at project submission.

## This readme is a guide to deploying your own instance of the [ItemCatalog-FSND project](https://github.com/GideonFlynn/ItemCatalog-FSND) on DigitalOcean.

First, I needed to be certain the app would run with PostgreSQL.

The adjustments needed to be made weren't that large: 
1. Change all 'NVARCHAR' columns in dbmodels.py to 'Text'
2. Make sure all paths are absolute and configured for a Linux OS
3. Make sure the items table's foreign keys reference unique columns
    - Before the shop and manufacturer tables had a foreign key on their 'id' column, now the foreign key for manufacturer is 'name'
   
    **This means that when specifying a manufacturer when creating an item, you type their name.**

4. Another change to be made was setting up a Postgres user and database, both named catalog, to enable testing of the app

    - When the database and user had been made, the last step was:

5. Changing `create_engine()` in catalog.py & dbmodels.py to `postgresql://catalog:catalog@localhost/catalog`

_This was a great time to test everything in the app and make sure nothing was broken. As you'd hope no breaking changes had been made and everything runs smoothly._

-------- 
_The following guide assumes you are redirecting your DNS provider's name servers to DigitalOcean._

Setting up DNS on DigitaOcean is done as follows:
- Go to the network tab
- Enter the domain
- Declare your favorite DNS records, mine are:
  - **A** record to gideonservice.me
  - **CNAME** record to make any prefix redirect to gideonservice.me
  - **NS** records to DigitalOcean
# Setting up a DigitalOcean Server

Go to digitalocean.com and create a droplet with Ubuntu 14.04 and wait for it to deploy.
### Configuring authorization
- Open a bash terminal locally
  - On Windows: install git and use git-bash
- Copy the temporary password from the email you received at droplet creation
```bash
ssh root@< server IP-address >
< temporary password >
adduser < user >
gpasswd -a < user > sudo
sudo cp /etc/sudoers.d/90-cloud-initusers /etc/sudoers.d/< user >
sudo nano /etc/sudoers.d/< user >
```
inside the sudoers.d/grader change the word `root` to `< user >`
  -  < user > can run commands with sudo privileges now!


- Locally run `ssh-keygen`, name the key by saving it to your default .ssh directory
- Locally run `cat ~/.ssh/< RSA-key-name >.pub` and copy it

**On the server as root user**
```bash
su < user >
cd  /home/< user >
mkdir .ssh
sudo nano .ssh/authorized_keys
```
(Paste the contents of < rsa-key-name >.pub then save and exit)
```bash
sudo chmod 644 .ssh/authorized_keys
sudo chmod 700 .ssh
exit (until current user is root)
```
---
- Run `sudo nano /etc/ssh/sshd_config`

  - Change the line `PermitRootLogin yes` to `PermitRootLogin no`
  
  - Change the line `PasswordAuthentication yes` to `PasswordAuthentication no`
  - Change the line `Port 22` to `Port 2222`
  - Save and exit
- Run `sudo service ssh restart`

### Logging in as < user >

Locally, open bash and run `ssh < user >@< server-IP > -p 2222 -i ~/.ssh/< rsa-key-name >`

## Software installation
Run the following commands:
```bash
sudo apt-get -y update
sudo apt-get -y install finger
sudo apt-get -y install python2.7 python-pip
sudo -H pip2 install --upgrade pip
sudo apt-get -y install postgresql postgresql-contrib
sudo apt-get -y install apache2 libapache2-mod-wsgi
sudo apt-get -y install python-dev
sudo apt-get -y install git
sudo apt-get -y update
sudo apt-get -y upgrade
sudo apt-get -y autoremove
```
**To confirm everything is working as it should so far: go to your browser, enter your servers IP-address and see if the default apache page is there. If nothing displays, run above commands again. If you still don't receive anything you should start over.**

## Setting up PostgreSQL
```bash
sudo su postgres
createuser --interactive -W -P < DB-username >
createdb -O < DB-owner > < DB-name >
exit
```
Whenever you make a change to the configuration of most services in Linux, you will have to restart them for the change to take effect. For now, use these commands:

- `sudo service postgresql restart`

- `sudo service ssh restart`

## Download and configure the application
*This is assuming apache2 is installed properly.*
```bash
cd /var/www/
sudo mkdir flaskapps
sudo cd /flaskapps
sudo mkdir catalog
sudo git clone https://github.com/GideonFlynn/ItemCatalog-FSND.git
sudo mv /ItemCatalog-FSND/* /catalog/
```
_To remove unused files/folders (and the .git folder), run these commands:_
```bash
cd /var/www/flaskapps/
sudo rm -r -f ItemCatalog-FSND/
```
### Setting up a virtual environment for Python
```bash
cd var/www/flaskapps/catalog
sudo pip install virtualenv
sudo virtualenv venv
source venv/bin/activate
sudo pip install -r requirements.txt
deactivate
```
**Then we'll enable the app, first run:**
```bash
sudo a2enmod wsgi
sudo nano /etc/apache2/sites-available/catalog.conf
```

**Then paste the following code into catalog.conf**
```
<VirtualHost *:80>
                ServerName < server-IP >
                ServerAlias < DNS >
                ServerAdmin < admin-email@example.com >
                WSGIScriptAlias / /var/www/flaskapps/catalog.wsgi
                <Directory /var/www/flaskapps/catalog/>
                        Order allow,deny
                </Directory>
                Alias /static /var/www/flaskapps/catalog/static
                <Directory /var/www/flaskapps/catalog/static/>
                        Order allow,deny
                        Allow from all
                </Directory>
                ErrorLog ${APACHE_LOG_DIR}/error.log
                LogLevel warn
                CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Save and exit
# Configuring the WSGI
```bash
sudo a2ensite catalog
cd /var/www/flaskapps/
sudo nano catalog.wsgi
```
**Paste the following code into /var/www/flaskapps/catalog.wsgi**
```python
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/flaskapps/catalog")

from catalog import app as application
application.secret_key = 'Add your secret key'
```
Save and exit

Run `cd /var/www/flaskapps/catalog`

**We want to make sure the /static/uploads/images folders are present.**

If not:
```bash
cd /static
sudo mkdir uploads
cd /uploads
sudo mkdir images
cd /var/www/flaskapps/catalog
```
**If/when the folder is present:**
```bash
cd /var/www/flaskapps/catalog
sudo chmod 777 -R static/
sudo service apache2 restart
```
## Setting up The Uncomplicated Firewall a.k.a ufw
_We waited to configure the firewall until now, so we're certain to be able to log in even if something goes wrong._
```bash
sudo ufw status
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 2222/tcp
sudo ufw allow www
sudo ufw allow ntp
sudo ufw enable (press 'y')
sudo ufw status
```
_Now your terminal should display this text:_
```bash
Status: active

To                         Action      From
--                         ------      ----
22                         ALLOW       Anywhere
2222/tcp                   ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
123                        ALLOW       Anywhere
22 (v6)                    ALLOW       Anywhere (v6)
2222/tcp (v6)              ALLOW       Anywhere (v6)
80/tcp (v6)                ALLOW       Anywhere (v6)
123 (v6)                   ALLOW       Anywhere (v6)
```
# Configuring the time zone 
- Run `sudo dpkg-reconfigure tzdata`
- Select 'None of these' with your arrow keys
- Select UTC

# Reboot!
*After initial configuration, it would be a good idea to reboot the system:*
- Run `sudo apt-get update` and `sudo apt-get -y dist-upgrade`.
- Run `reboot` 

- Login with ssh
  `ssh < user >@< server-IP > -p 2222 -i ~/.ssh/< rsa-file-name >`
#### In your browser, go to the server name/alias you defined in catalog.conf and enjoy!
