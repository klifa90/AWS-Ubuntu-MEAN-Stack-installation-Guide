# AWS-Ubuntu-MEAN-Stack-installation-Guide
Installation and basic configuration for AWS Ubuntu instance for full MEAN stack.
In this guide I will show you how to install and configure everything we need to run Mongo (database), NodeJS running with Express (backend/api) and host an Angular site (frontend/web/backoffice).

**IMPORTANT: This guide assumes you have already created your AWS account, created a new aws ubuntu instance and are already connected via SSH to your instance. Every command should be run with SUDO.**

###**1) Install npm and nodejs**

We should refresh our local package index first, and then install from the repositories:

    sudo apt-get update
    sudo apt-get install nodejs
    sudo apt-get install npm

###**2) Install PM2 (similar to forever, it will run your script continuously)**

    sudo npm install pm2 -g
    sudo ln -s "$(which nodejs)" /usr/bin/node (if necessary - read below)

Ubuntu install by default ‘node’ as ‘nodejs’, many programs/scripts use the ‘node’ command which won’t be found. Like PM2. This line will change ‘node’ calls to ‘nodejs’ automatically.

###**3) Install NGINX and create working directories**

    sudo apt-get install nginx
    sudo service nginx start

nginx should create /www/html inside var directory. Then, create inside /var/www/html the folders we will be working with (web for the site, backend, backoffice, etc.)

    sudo mkdir web, sudo mkdir backend, sudo mkdir backoffice

###**4) Configure sites-enabled/default and nginx.conf for whatever we need for our sites. We can also enabled sub-domains**

nginx folder is in etc/nginx directory

Check DEFAULT file for site, backoffice and api configuration example with https

Nginx.conf file doesn’t need a lot of configuration. You should add/modify this line 

    client_max_body_size 5M

to whatever you need. This is the size limit for requests to the server. If you need to upload files for example, you might need to expand this limit. In this example size is 5Mb.

Also check that SSL is enabled and default

    #SSL Settings
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
    ssl_prefer_server_ciphers on;

Check the "default" and "nginx.conf" files in this repository if you have any additional doubt

###**5) Run PM2 and add it to instance startup**

First you should upload all your backend files. The best approach for this is simple add your git repository to the folder and just pull. (git is already installed on ubuntu by default)
Then to start run 

    sudo pm2 start index.js (inside your backend directory, index.js is your initial script)
    sudo pm2 startup
    sudo pm2 save
    sudo pm2 startup (again to check everything is fine)

###**6) Free SSL/HTTPS for your domains/subdomains and automatic renewalas described in: https://letsencrypt.org/ and https://certbot.eff.org/** 

    sudo apt-get install letsencrypt 

IMPORTANT: For automatic configuration and renewal, the site/domain has to be accessible via http before running this next command. You might need to edit your sites-enabled to listen on port 80 instead of 443 (http instead of https) and comment with “#” the ssl lines. And afterwards leave it with https again. If you don’t want to do this you can run the second command below (--standalone command).

    sudo letsencrypt certonly --webroot -w /var/www/html/web -d mysite.com -d www.mysite.com

Here a screen will appear and you will have to accept ToS.

You should also run this command for every other subdomain you have, for example the backoffice subdomain

    sudo letsencrypt certonly --webroot -w /var/www/html/backoffice -d admin.mysite.com

For api and if the command above fails, this is the standalone command, preferably run from 

    /var/www/html/backend directory
    sudo letsencrypt certonly --standalone -d api.mysite.com

check automatic renew

    letsencrypt renew --dry-run --agree-tos


Then create cron to run 2 times a day that will execute the following command.
letsencrypt renew

For this, there are many ways to do it, we will do it with crontab.

    sudo crontab -e

Add this lines at the end

    30 2 * * 1 letsencrypt renew >> /var/log/le-renew.log
    35 2 * * 1 /etc/init.d/nginx reload


###**7) Install and set up MONGODB**

First we should add the key to ubunto so apt-get command can get and install the package.

    sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 0C49F3730359A14518585931BC711F9BA15703C6


Then add mongodb repo url for ubuntu (you should get your ubuntu version and get the corresponding latest url, this is for xenial 16.04)

    echo "deb [ arch=amd64,arm64 ] http://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.4.list

Now we can finally install mongo

    sudo apt-get install mongodb-org

Run the service

    sudo service mongod start

Check that everything is ok by checking /var/log/mongodb/mongod.log and finding this line 
[initandlisten] waiting for connections on port <port>


Check/add database directories have permission for mongodb user (yes, mongo has its own user for the instance, so we have to give that user permissions)

    chown -R mongodb:mongodb /var/log/mongodb/mongod.log
    chown -R mongodb:mongodb /var/lib/mongodb/

Add it on boot

    sudo systemctl enable mongod.service

**Mongod.service:** This is the file that will run when starting mongo, no configurations should be made here, just check the directories name match and user and group is set to mongodb.

    sudo nano /lib/systemd/system/mongod.service

**Mongod.conf:** In this file, bindIP is to specify where we will allow incoming connections to database, by default it will only allow connections from localhost (same machine). 
The security section is to allow authentication when connecting to a database, we should definitely edit this after we create database users. 

    sudo nano /etc/mongod.conf

Now connect to mongo shell and create super admin user

    mongo
    use admin
    db.createUser(
      {
        user: "myUserAdmin",
        pwd: "abc123",
        roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
      }
    )

Then, to connect using this user you should run this command

    mongo --port 27017 -u "myUserAdmin" -p "abc123" --authenticationDatabase "admin"

or just simple connect and run this command

    db.auth("myUserAdmin", "abc123" )

Now we should create our database we will use for our application, with the user to access it and it permissions. We will create a single user which will be the owner of this database

    use mynewdatabasename (this will create and enter the new database)
    db.createUser(
      {
        user: "username",
        pwd: "astrongpassword",
        roles: [ “dbOwner”]
      }
    )

Now that we have users setted up, we can exit mongo shell and edit mongo conf to allow authentication and external connections if we need to. 

    sudo nano /etc/mongod.conf
    Search “bindIp” and comment it with #
    Search “security” and make sure it looks like this (uncomment and add auth line)
    security:
      authorization: 'enabled'

Test that everything is ok by trying to log into our newly created user and insert something on the database. Use authentication command above on mynewdatabasename and insert the following collection

    db.foo.insert( { x: 1, y: 1 } )


###**Ulimit settings**
Finally, for production instance, check the following Ulimit settings in your AWS instance. You might only need to change “open files” setting.

    sudo su
    ulimit -n 999999

At your production environment you may not have problems with system limits. The following thresholds and settings are particularly important for mongod and mongos deployments:

    -f (file size): unlimited
    -t (cpu time): unlimited
    -v (virtual memory): unlimited
    -n (open files): 999999
    -m (memory size): unlimited
















