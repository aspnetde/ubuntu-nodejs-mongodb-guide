# Setting up Ubuntu Server for running a single Website with Node.js and MongoDB

This little guide shows how to set up an Ubuntu Server that is dedicated to run a single website with Node.js and MongoDB. If you are looking for a more generic solution to run multiple websites on a single server, take a look at the [Node.js Web Server Guide](https://github.com/aspnetde/nodejs-webserver-guide). It provides some more details to security aspects which don't matter if there is only one application running.

## Create your Droplet (DigitalOcean only)

I won't tell you how to create a Droplet, because it seems self-explaining to me. If you need any help with this, this little tutorial isn't the thing you should read anyway, at least yet ;-).

## Create a User called www

You could run all your stuff as root, but I don't think that's a good idea. So connect to your only just created server and log in via root:

	ssh root@{ip-address}
	
Next, create a the www user:

	adduser www
	
Now provide root privilige, (Other than the root account the www user won't run with these priviliges all the time, but it could when requested, what will be necessary at least during the installation process.)

Call `visudo` and add the following line right below the root's line:
	
	www ALL=(ALL:ALL) ALL

Now `exit` your ssh connection and re-connect as www.

## Set up SSH Key for www

	cat ~/.ssh/id_rsa.pub | ssh www@{ip-address} "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"

	
At this point you should be requested to provide the password of the www user at login for the last time. `exit` and reconnect – now you should be authenticating via SSH Key.

	ssh www@{ip-address}
	
## Install the required software

### Make Tools

The make tools are essential to build some npm packages and other stuff. So it’s generally a good idea to install them early.

	sudo apt-get install gcc make build-essential

(If the installation of build-essential fails, see Misc/Missing packages section).

### nginx

	sudo apt-get install nginx
	
Once the setup of nginx is complete, you should be able to call http://{server_ip} and see the default page with the “Welcome to nginx!” headline.

Also make sure the server starts automatically after booting the system (Should be enabled by default):

	sudo update-rc.d nginx defaults
	
### Node.js

If not installed with the initial creation of your droplet (DigitalOcean only; workes just fine!), use this:

	wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.14.0/install.sh | bash

	# Refresh Path
	source ~/.profile

	# Use latest Node.JS version
	nvm install v0.11.13
	
	# Make it default
	nvm use default v0.11.13

### Bower

	sudo npm install bower -g

### PM2

PM2 helps to run the node application by logging errors, restarting after crashing etc.

	sudo npm install pm2 -g

### Glances

Glances can be used to monitor the overall state of the server.

	sudo apt-get install python-pip build-essential python-dev
	sudo pip install Glances
	sudo pip install PySensors

### Git

	sudo apt-get install git

### Zip

	sudo apt-get install zip

### MongoDB

For a detailed explanation see [http://docs.mongodb.org/manual/tutorial/install-mongodb-on-ubuntu/](http://docs.mongodb.org/manual/tutorial/install-mongodb-on-ubuntu/).

Install the database service:

	sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
	echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' | sudo tee /etc/apt/sources.list.d/mongodb.list
	sudo apt-get update
	sudo apt-get install -y mongodb-org

Pin the current version:

	echo "mongodb-org hold" | sudo dpkg --set-selections
	echo "mongodb-org-server hold" | sudo dpkg --set-selections
	echo "mongodb-org-shell hold" | sudo dpkg --set-selections
	echo "mongodb-org-mongos hold" | sudo dpkg --set-selections
	echo "mongodb-org-tools hold" | sudo dpkg --set-selections

### Add the website’s directories

Websites are organized as follows:

Directory | Path
------------ | -------------
Root | /var/www
Git repository	 | /var/www/repo
Website root | /var/www/www

	cd /var
	sudo mkdir www
	sudo chown www www
	cd www
	mkdir repo && mkdir www
	
### Create a Git repository

In `/var/www/repo` run

	git init --bare

### Add the Git deployment hook

This hook is used to deploy changes made to the master repository. It can be customized for each website depending on the specific needs.

Go to `/var/www/repo/hooks` and create a new file called “post-receive”:

	vi post-receive
	
Add the following commands to it:

	#!/bin/bash

	# Make this executable: chmod +x post-receive

	PREPARATION_DIR="/var/www/repo/$(uuidgen)"
	WEBSITE_ROOT="/var/www/www"
	PM2_APP_NAME="website-com"

	echo "Deployment started"

	read oldrev newrev branch

	if [[ $branch =~ .*/master$ ]];
	then
    	echo "Master received. Deploying to production..."

    	# Creates a temporary working directory
	    mkdir $PREPARATION_DIR

    	# Checks out the master from the repository
	    GIT_WORK_TREE="$PREPARATION_DIR" git checkout -f

    	# Installing all npm and bower modules/packages
	    cd $PREPARATION_DIR
	    npm install
	    bower install

	    # Removes all files in the Website's root
    	cd $WEBSITE_ROOT
		rm -rf *

		# Copies all files over
		cd $PREPARATION_DIR
		cp -r . $WEBSITE_ROOT

		# Restart the Website via PM2
		pm2 restart $PM2_APP_NAME

		# Removes the preparation directory
		rm -R $PREPARATION_DIR
	else
    	echo "$branch successfully received. Nothing to do: only the master branch may be deployed on this server."
	fi

	echo "Deployment finished"

Remember the value of **PM2_APP_NAME** and use it as an identifier for your pm2 application later.

After saving, make the script executable:

	chmod +x post-receive

### Push your application to your repository

By cloning your deployment repository on your local development machine and pushing the first version to the master branch, you should receive a fresh version at `/var/www/www` (Check with `ls –l`).

	git clone ssh://{username}@{ipaddress}/var/www/repo website-com

Make sure the user you’re connecting with has the necessary rights to run the Git repository. It’s recommended to connect with the user you have just created before to run the website, because he/she has the necessary access rights.

### Set up PM2

#### Run the startup script

To start pm2 with the system:

	pm2 startup ubuntu
	
PM2 will tell you, you have to run this command as root, and print the full command to execute, for example:

	sudo env PATH=$PATH:/usr/local/bin pm2 startup ubuntu -u www
	
Run it :-).

#### Start your application 

	cd /var/www/website-com/www/
	pm2 start app.js --name "website-com"

If everything works PM2 reponds with `Process {nameofstarting.js}` launched. Wait a few seconds and use

	pm2 list

for a fresh status update. For more information see

	pm2 help

### Configure the website in nginx

We’re using a single configuration, which can be found at:

	sudo vi /etc/nginx/sites-available/default

nginx is running as a reverse proxy to handle all the public stuff on port 80 for us. It then passes all the traffic we want to to our node application.

	server {
    	listen 80;

	    server_name your-domain.com;

    	location / {
        	proxy_pass http://localhost:{YOUR_PORT};
	        proxy_http_version 1.1;
    	    proxy_set_header Upgrade $http_upgrade;
        	proxy_set_header Connection 'upgrade';
	        proxy_set_header Host $host;
    	    proxy_cache_bypass $http_upgrade;
	    }
	}

After saving the configuration, use

	sudo service nginx reload

to tell the server it should use it.

# Backup

## Directory structure

Directory | Path
------------ | -------------
Backup Root | /var/backup
Website Backups | /var/backup/www
MongoDB Backups | /var/backup/mongo
nginx Backups | /var/backup/nginx

	cd /var
	sudo mkdir backup
	sudo chown www backup
	cd backup

## Backup Scripts

### MongoDB

Save the following shell script as `/var/backup/create-backup-for-mongo` and make it executable:

	#!/bin/bash

	echo "Mongo Backup started"

	BACKUP_TARGET_ROOT="/var/backup/mongo"
	CURRENT_BACKUP_TARGET="$BACKUP_TARGET_ROOT/$(uuidgen)"

	# Remove all but the latest 7 backups
	cd $BACKUP_TARGET_ROOT
	rm -rf `ls -t | tail -n +7`

	# Back up all the databases to a new directory
	mongodump -o $CURRENT_BACKUP_TARGET --authenticationDatabase admin

	zip -r "$(uuidgen).zip" $CURRENT_BACKUP_TARGET
	rm -rf  $CURRENT_BACKUP_TARGET

	echo "Mongo Backup finished"

### Websites

Create the script `/var/backup/create-backup-for-www` and make it executable:

	#!/bin/bash

	echo "WWW Backup started"

	BACKUP_SOURCE="/var/www"

	BACKUP_TARGET_ROOT="/var/backup/www"
	CURRENT_BACKUP_TARGET="$BACKUP_TARGET_ROOT/$(uuidgen)"
	
	cd $BACKUP_TARGET_ROOT
	rm -rf `ls -t | tail -n +7`

	# Back up all the websites to a new directory
	rsync -a -E -c --stats $BACKUP_SOURCE $CURRENT_BACKUP_TARGET

	zip -r "$(uuidgen).zip" $CURRENT_BACKUP_TARGET
	rm -rf  $CURRENT_BACKUP_TARGET

	echo "WWW Backup finished"
	
### nginx

Create the script `/var/backup/create-backup-for-nginx` and make it executable:

	#!/bin/bash

	echo "nginx Backup started"

	BACKUP_SOURCE="/etc/nginx"

	BACKUP_TARGET_ROOT="/var/backup/nginx"
	CURRENT_BACKUP_TARGET="$BACKUP_TARGET_ROOT/$(uuidgen)"

	# Remove all but the latest 7 backups
	cd $BACKUP_TARGET_ROOT
	rm -rf `ls -t | tail -n +7`

	rsync -a -E -c --stats $BACKUP_SOURCE $CURRENT_BACKUP_TARGET

	zip -r "$(uuidgen).zip" $CURRENT_BACKUP_TARGET
	rm -rf  $CURRENT_BACKUP_TARGET

	echo "nginx Backup finished"

## Transfer

There are many ways to transfer these backup files to another server, I have chosen the way to use rsync over SSH.

### Set up SSH 

First create a local key without a password:

	ssh-keygen -f ~/.ssh/id_rsa -q -P ""

Now get the public key and copy it:

	vi ~/.ssh/id_rsa.pub
	
On your backup server add the public SSH key of your web server. If you did not set up SSH before, do it as follows:

	mkdir ~/.ssh
	chmod 0700 ~/.ssh
	touch ~/.ssh/authorized_keys
	chmod 0644 ~/.ssh/authorized_keys
	
	# Paste the public key here:
	vi ~/.ssh/authorized_keys


### Add the Transfer Script

Create a script that combines all backup actions and that finally transfers everything from the current backup folder to the backup server. Save that script as `/var/backup/create-and-transfer-backups` and make it executable.

	#!/bin/bash

	echo "Global Backup started"

	# Important: use absolute paths to be independent of the user context
	/var/backup/create-backup-for-mongo
	/var/backup/create-backup-for-www
	/var/backup/create-backup-for-nginx

	rsync -avz -e "ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null" --progress /var/backup www-backup@{ip-of-your-backup-server}:/var/backup

	echo "Global Backup finished"

### Schedule backup

	sudo vi /etc/crontab 

Set:


	# m  h  dom mon dow user command
	10 14 *   *   *   root bash /var/backup/create-and-transfer-backups

(Runs the backup every day at 2:10 pm.)

#### Troubleshooting

If it doesn't work, check your timezone. If it is set wrong, you can change it easily (Ubuntu):

	sudo dpkg-reconfigure tzdata

Now restart cron to apply the new setting:
	
	sudo service cron restart
