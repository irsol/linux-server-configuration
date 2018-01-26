# Linux server configuration

Final project of Udacity Full Stack Web Developer Nanodegree

The goal of the project is to deploy [Item Catalog Application](https://github.com/irsol/item-catalog-app) as Amazon Lightsail.

The server is available by this url: http://13.59.27.59/

## Steps

1. Create an account on AWS and launch an instance with Ubuntu operational system

2. Change ssh port from 22 to 2200

	1. Connect to the server
		```
		chmod 400 LightsailDefaultPrivateKey-us-east-2.pem
		ssh ubuntu@13.59.27.59 -i LightsailDefaultPrivateKey-us-east-2.pem
		```

	2. Edit file _/etc/ssh/sshd_config_
		```
		Port 2200
		PermitRootLogin no
		PasswordAuthentication yes
		```

	3. Restart ssh server: 
		```
		service sshd restart
		```

	4. Change ports in [firewall](https://lightsail.aws.amazon.com/ls/webapp/us-east-2/instances/item-catalog/networking)

3. Create a user **grader**

	1. Create a new account with root permissions
		```
		sudo su -
		adduser grader
		usermod -aG sudo grader

		mkdir /home/grader/.ssh
		chown grader:grader /home/grader/.ssh
		chmod 700 /home/grader/.ssh
		cp /root/.ssh/authorized_keys /home/grader/.ssh/
		chown grader:grader /home/grader/.ssh/authorized_keys
		chmod 644 /home/grader/.ssh/authorized_keys
		```

	2. Create ssh keys and copy to the user folder
		```
		ssh-keygen -f ssh_keys/grader
		cat ssh_keys/grader.pub | ssh grader@13.59.27.59 -p 2200 "cat >>  ~/.ssh/authorized_keys"
		```

	3. Disable login with password in file /etc/ssh/sshd_config
		```
		PasswordAuthentication no
		```

4. Configure Linux

	1. Install latest software
		```
		ssh grader@13.59.27.59 -p 2200 -i ssh_keys/grader
		sudo apt-get -qy upgrade
		sudo apt-get -qy update
		```

	2. Set timezone to UTC and locale
		```
		sudo timedatectl set-timezone UTC

		echo 'LC_ALL="en_US.UTF-8"' >> /etc/environment
		source /etc/environment
		export LC_ALL
		```

	3. Install apache2, postgres and python
		```
		sudo apt-get install apache2
		sudo apt-get install libapache2-mod-wsgi-py3
		sudo apt-get install python3-pip

		sudo pip3 install --upgrade pip
		sudo pip3 install flask
		sudo pip3 install sqlachemy
		sudo pip3 install oauth2client
		sudo pip3 install psycopg2

		sudo apt -qy install postgresql python3-psycopg2 libpq-dev
		```

	4. Setup a database
		```
		sudo -u postgres createuser -P catalog
		sudo -u postgres createdb -O catalog catalog
		```

5. Deploy project

	1. Clone project to _/var/www/_
		```
		cd /var/www/
		git clone https://github.com/irsol/item-catalog-app.git
		```

	2. Create **server.wsgi** file
		```
		import sys
		import logging

		logging.basicConfig(stream=sys.stderr)
		sys.path.insert(0, "/var/www/item-catalog-app/")

		from server import app as application
		```

	3. Update _/etc/apache2/sites-enabled/000-default.conf_
		```
		<VirtualHost *:80>
			WSGIScriptAlias / /var/www/item-catalog-app/server.wsgi

			<directory /var/www/item-catalog-app/>
				WSGIApplicationGroup %{GLOBAL}
				Require all granted
			</directory>

			ErrorLog ${APACHE_LOG_DIR}/error.log
			CustomLog ${APACHE_LOG_DIR}/access.log combined
		</VirtualHost>	
		```

	4. Change database connection from sqlite to postgres in database_setup.py
		```
		engine = create_engine('postgresql://catalog:<password>@localhost:5432/catalog')
		```

	5. Copy **client_secrets.json** to _/var/www/item-catalog-app_

	6. Restart apache:
		```
		sudo apache2ctl restart
		```
