


# Linux Server Configuration

#### Website link: http://ec2-54-93-109-154.eu-central-1.compute.amazonaws.com/

* **Public IP Address:** ## 54.93.109.154
* **Accessible SSH port:** 2200

## Steps to Configure Linux server

#### 1. Create new user
  
  * Add user grader
  
```
$ sudo adduser grader
```
  * Give sudo access to grader
  
```
$ sudo nano /etc/sudoers.d/grader
```
  * Add following line to this file
  
```
grader ALL=(ALL) NOPASSWD:ALL
```
  * To prevent the "sudo: unable to resolve host" error
  
   i. Edit the hosts file:
   
```
$ sudo nano /etc/hosts
```
   ii. Add the host:
```
$ 127.0.1.1 ip-172-26-0-49
```

#### 2. Configure authentication for grader user

   i. Copy public key from root *$ sudo cat /home/ubuntu/.ssh/authorized_keys*
   
   ii. Place the public key from root to grader:
   
```
$ sudo touch /home/grader/.ssh/authorized_keys
$ sudo nano /home/grader/.ssh/authorized_keys
```
     
#### 3. Enforce key-based authentication
  * Run `$ sudo nano /etc/ssh/sshd_config`.
  * Find the **PasswordAuthentication** line and edit it to no.
  * Save the file.
  * Run `$ sudo service ssh restart` to restart the service.

#### 4. Change the SSH port from 22 to 2200
  * Find the **Port line** in the same file above, i.e */etc/ssh/sshd_config* and edit it to 2200.
  * Save the file.
  * Run `$ sudo service ssh restart` to restart the service.
  
#### 5. Disable login for root user
  * Find the **PermitRootLogin** line in the same file above, i.e */etc/ssh/sshd_config* and edit it to no.
  * Save the file.
  * Run `$ sudo service ssh restart` to restart the service.

Now you can login into remote VM through SSH with following command:
```
$ ssh grader@54.93.109.154 -p 2200 -i ~/.ssh/lightsail_key.pem
```

#### 6. Configure local timezone to UTC
  * Change the timezone to UTC using following command: `$ sudo timedatectl set-timezone UTC`.
  * You can also open time configuration dialog and set it to UTC with: `$ sudo dpkg-reconfigure tzdata`.
  * Install ntp daemon ntpd for a better synchronization of the server's time over the network connection:
  
```
   $ sudo apt-get install ntp
```

#### 7. Update all currently installed packages
  * `$ sudo apt-get update`.
  * `$ sudo apt-get upgrade`.

#### 8. Configure the Uncomplicated Firewall (UFW)
```
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow 2200/tcp
$ sudo ufw allow www
$ sudo ufw allow ntp
$ sudo ufw enable
```

#### 9. Install and Configure Apache2, mod-wsgi and Git
```
$ sudo apt-get install apache2 libapache2-mod-wsgi git
```
 * Enable *mod_wsgi*:
 
```
$ sudo a2enmod wsgi
```

#### 10. Install and configure PostgreSQL
  * Installing PostgreSQL Python dependencies:
  
```
$ sudo apt-get install libpq-dev python-dev
```
  * Installing *PostgreSQL*:

```
$ sudo apt-get install postgresql postgresql-contrib
```

  * Login as *postgres* User (Default User), and get into PostgreSQL shell:
```
$ sudo su - postgres
$ psql
```

Create a new user named *movie_catalog*: 

`# CREATE USER movie_catalog WITH PASSWORD 'movie_catalog';`

Create a new database named *movie_catalog: 

`# CREATE DATABASE movie_catalog WITH OWNER movie_catalog;`

Connect to the database *catalog* : 

`# \c movie_catalog`

Revoke all rights: 

`# REVOKE ALL ON SCHEMA public FROM public;`

Lock down the permissions only to user *movie_catalog *: 

`# GRANT ALL ON SCHEMA public TO movie_catalog;`

Log out from PostgreSQL: `# \q`. Then return to the *grader* user: `$ exit`

 Inside the application.py, database_setup.py and add_movies.py changed from 

```
   engine = create_engine('postgresql://movie_catalog :movie_catalog @localhost/movie_catalog')
```

#### 11. Install Flask and other dependencies

```
$ sudo apt-get install python-pip
$ sudo pip install Flask
$ sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils
$ sudo pip install requests
```

#### 12. Clone the item-catalog app from Github

  Clone the **item-catalog** to the catalog directory `/var/www/`:

```
$ git clone https://github.com/yasir-albardawil/item-catalog.git catalog
```
  Change the owner of the directory *catalog*
```
$ sudo chown -R grader:grader /var/www/catalog
```

  Change the branch of repo **item-catalog**  to **deployment**:
```
$ cd catalog && git checkout deployment
```

  Make an application.wsgi file  to serve the application over the mod_wsgi. with content:

```
$ touch application.wsgi && nano application.wsgi
```

```
import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from application import app as application
application.root_path = '/var/www/catalog'
application.secret_key = 'super_secret_key'
```
 * Inside *application .py - database_setup.py - add_movies.py*  database connection is now performed with:
```
engine = create_engine('postgresql://movie_catalog:movie_catalog@localhost/movie_catalog')
```
  * Run the *database_setup.py* and *add_movies.py* once to setup database with dummy data:
  
```
$ python database_setup.py
$ python add_movies.py
```

#### 13.  Create virtual host configuration:

```
$ sudo nano /etc/apache2/sites-available/catalog.conf
```

```
<VirtualHost *:80>
   ServerName 54.93.109.154
   ServerAdmin yasir.s.albardawil@gmail.com
   SetEnv OAUTHLIB_INSECURE_TRANSPORT 1
   WSGIDaemonProcess catalog user=ubuntu group=ubuntu threads=2
   WSGIScriptAlias / /var/www/catalog/application.wsgi
   <Directory /var/www/catalog>
     WSGIProcessGroup catalog
     WSGIApplicationGroup %{GLOBAL}
     <IfVersion < 2.4>
        Order allow,deny
        Allow from all
     </IfVersion>
     <IfVersion >= 2.4>
        Require all granted
      </IfVersion>
   </Directory>
   Alias "/static/" "/var/www/catalog/static/"
   <Directory /var/www/catalog/static/>
     <IfVersion < 2.4>
        Order allow,deny
        Allow from all
     </IfVersion>
     <IfVersion >= 2.4>
        Require all granted
      </IfVersion>
   </Directory>
   ErrorLog ${APACHE_LOG_DIR}/error.log
   LogLevel info
   CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

Enable the new site, disable the default site, and reload  *apache2*.
```
$ sudo a2ensite catalog
$ sudo a2dissite 000-default
$ sudo rm 000-default
$ sudo service apache2 reload
```

### 14. Fix gp_client_secrets.json not found
add the folowing code inside application.py
```
CLIENT_ID = json.loads(
    open('/var/www/catalog/gp_client_secrets.json', 'r').read())['web']['client_id']
```
```
oauth_flow = flow_from_clientsecrets('/var/www/catalog/gp_client_secrets.json', scope='')
```
#### 18. In order to install packages *kept back* you have to run:
```
sudo apt-get update && sudo apt-get dist-upgrade
```

#### 19. Restart Apache to launch the app
```
$ sudo service apache2 restart
```
#### 20. List of third party resources used

 * https://hk.saowen.com/a/0a0048ca7141440d0553425e8df46b16cdf4c13f50df4c5888256393d34bb1b9
* https://mudspringhiker.github.io/deploying-a-flask-web-app-on-lightsail-aws.html
* https://www.youtube.com/watch?v=kDRRtPO0YPA&t=223s
* https://www.youtube.com/watch?v=Ta7HAvu-GPs&t=954s
* https://www.youtube.com/watch?v=iVGtJOC71Fw
* http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/
* https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps
* https://pypi.org/
