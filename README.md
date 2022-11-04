# Multi-Tier Application Deployment

![v-profile banner](https://user-images.githubusercontent.com/99980305/199944530-e15176c5-9c17-4806-bc45-2b0c79e74d47.png)

This application was inspired by devopshydclub repository found [here](https://github.com/devopshydclub/vprofile-project)


You can see the most used languages, in this project, below:

![Your Repository's Stats](https://github-readme-stats.vercel.app/api/top-langs/?username=fahimtq1&theme=blue-green)

## Project brief

The scenario is of a DevOps engineer who has been given a multi-tier application stack and has been tasked with deploying the application on a local machine. 

The application deals with a variety of services: web services, database services, application services etc.

The main challenge facing the DevOps engineer, regarding the local setup, is that the local setup is complex and time consuming. To solve this problem, the setup provisioning will be automated with Bash scripts and Virtual Machines will be configured with the use of Infrastructure as Code tools. 

First, the DevOps engineer will manually configure and provision the Virtual Machines, ready for application deployment. This is to ensure that the architecture network is properly configured, with any debugging handled before the automation step.

Finally, the entire setup will be automated. 

## Application architecture

![vprofile-project-architecture](https://user-images.githubusercontent.com/99980305/199768482-3bb654c1-8a40-4352-8e86-49b4ae50875f.png)

### Services

- **NGINX**- a web service used for load balancing and reverse proxying
- **Tomcat**- an application service used to run JAVA server pages that are based on web applications
- **RabbitMQ**- a message broker/queuing agent service (used as a dummy service in this project)
- **Memcached**- database caching service
- **MySQL**- SQL database

## Tools

- **Git Bash**
- **Oracle VM Virtualbox**
- **Vagrant**
    - Vagrant has a few necessary commands that should be ran in the location of the Vagrantfile- `cd vprofile-project/Manual_provisioning`
        - `vagrant plugin install vagrant-hostmanager`
        - `vagrant plugin install vagrant-vbguest`

## Manual setup

### Vagrantfile

The Vagrantfile automates the virtualisation process and the configuration of each Virtual Machine. In this project, five Virtual Machines will be created, and these are outlined in the Vagrantfile:

![vprofile-vagrantfile](https://user-images.githubusercontent.com/99980305/199950428-74b80312-af86-4b62-a9c6-1d97a8a121b3.png)

### MySQL- db01

- `vagrant shh db01`
- `sudo -i`- switches to root user
- `yum update -y`- good practice to patch bugs in the OS before provisioning
- `yum install epel-release -y`- installing extra required packages
- `DATABASE_PASS='admin123`- set temporary environment variable
- `echo DATABASE_PASS`- calls value of key
- `vi /etc/profile`- editing this file to make the variable permanent
    - add `DATABASE_PASS='admin123` to the file
    - `source /etc/profile`- read and executes the file content
- `yum install git mariadb-server -y`- installs MariaDB package
- `systemctl start mariadb`- starts the service
- `systemctl enable mariadb`
- `mysql_secure_installation`- setup MariaDB database
- The following questions will appear, ensure to input the correct responses as shown below (password is admin123 for this project):

```
Set root password? [Y/n] Y New password:
Re-enter new password:
Password updated successfully! Reloading privilege tables..
... Success!

By default, a MariaDB installation has an anonymous user, allowing anyone to log into MariaDB without having to have a user account created for
them. This is intended only for testing, and to make the installation go a bit smoother. You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] Y
... Success!

Normally, root should only be allowed to connect from 'localhost'. This ensures that someone cannot guess at the root password from the network.
Disallow root login remotely? [Y/n] n
... skipping.

By default, MariaDB comes with a database named 'test' that anyone can access. This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] Y
-	Dropping test database...
... Success!
-	Removing privileges on test database...
... Success!

Reloading the privilege tables will ensure that all changes made so far will take effect immediately.

Reload privilege tables now? [Y/n] Y
... Success!
```

Setting up the database:

- `git clone -b local-setup https://github.com/devopshydclub/vprofile-project.git`- clone original project repository
- `mysql -u root -p"$DATABASE_PASS" -e "create database accounts"` - creates a database named accounts
-  `mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'app01' identified by 'admin123'"`- grants all privileges to admin user in app vm with password admin123
- `mysql -u root -p"$DATABASE_PASS" accounts < src/main/resources/db_backup.sql` - executes sql file with given file path (depending on location)
- `mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"`- clears internal caches used my MariaDB

Checking the database:
- `mysql -u root -p"$DATABASE_PASS`- login to database
- `show databases;`
- `use accounts;`
- `show tables;` - sql query has initialised database with tables shown

Starting the firewall and allowing MariaDB to access from port no. 3306 to establish a connection between the MariaDB server and this application
- `systemctl start firewalld`
- `systemctl enable firewalld`
- `firewall-cmd --get-active-zones`
- `firewall-cmd --zone=public --add-port=3306/tcp --permanent`
- `firewall-cmd --reload`
- `systemctl restart mariadb`

### Memcached- mc01

- `vagrant ssh mc01`
- `sudo -i`
- `yum update -y`
- `yum install epel-release -y`

Install, start & enable memcache on port 11211:

- `yum install memcached -y`
- `systemctl start memcached`
- `systemctl enable memcached`
- `memcached -p 11211 -U 11111 -u memcached -d` - listen on TCP port and UDP port respectively
- `ss -tunlp | grep 11211`- validate port is in use

Starting the firewall and allowing the port 11211 to access memcached

- `systemctl enable firewalld` 
- `systemctl start firewalld`
- `systemctl status firewalld`
- `firewall-cmd --add-port=11211/tcp --permanent` 
- `firewall-cmd --reload`
- `memcached -p 11211 -U 11111 -u memcache -d`

### RabbitMQ- rmq01

- `vagrant ssh rmq01`
- `sudo -i`
- `yum update -y`
- `yum install epel-release -y`

Install dependencies (use sudo at the start if commands don't work):

- `yum install wget -y`- installs package that allows downloads from the web
- `cd /tmp/`- navigate to this directory as the location for the following download
- `wget http://packages.erlang-solutions.com/erlang-solutions-2.0-1.noarch.rpm`- download erlang rpm file
- `rpm -Uvh erlang-solutions-2.0-1.noarch.rpm` - install rpm
- `yum -y install erlang socat` - install erlang and socat

Install RabbitMQ Server:

- `curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | sudo bash ` -download shell script and execute simultaneously
- `sudo yum install rabbitmq-server -y` - install RabbitMQ server package
- `systemctl start rabbitmq-server`
- `sudo systemctl enable rabbitmq-server`

Configuration change:

- `sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'`- echos output into specified file
- `sudo rabbitmqctl add_user test test`- creates user in the RabbitMQ server with username and password
- `sudo rabbitmqctl set_user_tags test administrator`- gives user admin privileges 

Restart RabbitMQ service:
- `systemctl restart rabbitmq-server`

Enabling the firewall and allowing port 25672 to access RabbitMQ permanently:

- `systemctl start firewalld`
- `systemctl enable firewalld`
- `firewall-cmd --get-active-zones`
- `firewall-cmd --zone=public --add-port=25672/tcp --permanent` 
- `firewall-cmd --reload`

### Tomcat- app01

- `vagrant ssh app01`
- `sudo -i`
- `yum update -y`
- `yum install epel-release -y`

Install dependencies:

- `yum install java-1.8.0-openjdk -y`
- `yum install git maven wget -y` -> Maven is a build automation tool for Java projects

Install and configure Tomcat:

- `cd /tmp/`
`wget https://archive.apache.org/dist/tomcat/tomcat-8/v8.5.37/bin/apache-tomcat-8.5.37.tar.gz`
`tar xzvf apache-tomcat-8.5.37.tar.gz`- extracts the Tomcat software from this tar file and `apache-tomcat-8.5.37` is the Tomcat directory
- `useradd --home-dir /usr/local/tomcat8 --shell /sbin/nologin tomcat`- adds the Tomcat home directory (tomcat8) and a Tomcat user
- `id tomcat`- switch to tomcat user
- `cp -r /tmp/apache-tomcat-8.5.37/* /usr/local/tomcat8/`- copy all data to tomcat home directory
- `chown -R tomcat.tomcat /usr/local/tomcat8 -R`- changes privileges given to root user to tomcat user
- `vi /etc/systemd/system/tomcat.service`- make this file to use systemctl to start/stop tomcat service
    - Paste the following into the file:

```
[Unit] Description=Tomcat After=network.target

[Service] User=tomcat
WorkingDirectory=/usr/local/tomcat8 Environment=JRE_HOME=/usr/lib/jvm/jre Environment=JAVA_HOME=/usr/lib/jvm/jre Environment=CATALINA_HOME=/usr/local/tomcat8 Environment=CATALINE_BASE=/usr/local/tomcat8 ExecStart=/usr/local/tomcat8/bin/catalina.sh run ExecStop=/usr/local/tomcat8/bin/shutdown.sh SyslogIdentifier=tomcat-%i

[Install]
WantedBy=multi-user.target
```

- `systemctl daemon-reload`- daemon in linux is a service that executes in the background
- `systemctl start tomcat`
- `systemctl enable tomcat`

Code build and deploy:

- `git clone -b local-setup https://github.com/devopshydclub/vprofile-project.git`
- `cd vprofile-project`
- `vi src/main/resources/application.properties`- this file is an important file for the application
    - Update this file with the backend server details

Run below command inside the repository (vprofile-project) to build the code:
 - `mvn install`- builds the artifact

Artifact can be found in the target directory in vprofile-project/vprofile-v2.war.

Deploy the artifact
- `systemctl stop tomcat
- `rm -rf /usr/local/tomcat8/webapps/ROOT*`- replace the default application in the Tomcat server with the artifact
- `cp target/vprofile-v2.war /usr/local/tomcat8/webapps/ROOT.war`
- `systemctl start tomcat`- extracts ROOT.war artifact into ROOT directory` 
- `chown tomcat.tomcat usr/local/tomcat8/webapps -R`- all permissions are given to the user tomcat
- `systemctl restart tomcat`


Enabling the firewall and allowing port 8080 to access Tomcat:

- `systemctl start firewalld`
- `systemctl enable firewalld`
- `firewall-cmd --get-active-zones`
- `firewall-cmd --zone=public --add-port=8080/tcp --permanent` 
- `firewall-cmd --reload`

### NGINX- web01

- `vagrant ssh web01`
- `sudo -i`
- `apt update && apt upgrade -y`- patch the OS

Install nginx
- `apt install nginx -y`

Create nginx.conf file with the below content:
- `vi /etc/nginx/sites-available/vproapp`- custom NGINX reverse proxy file
    - Paste the below content into the file to allow the frontend to listen on port 80 and route the request to the app01 server on port 8080

![vprofile-nginxconf](https://user-images.githubusercontent.com/99980305/199947649-7ca173eb-26c3-476e-8fa8-a04a17c1d694.png)

Remove default files:

- `rm -rf /etc/nginx/sites-available/default`
- `rm -rf /etc/nginx/sites-enabled/default`
- `ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp`
- `systemctl restart nginx`


### Starting the application

- `ifconfig `- to find the IP address needed
- Type the IP address into your browser and see your application running!


## Automatic setup
