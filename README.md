# AWS Three Tier Web Architecture Workshop

## Architecture Overview
![Architecture Diagram](https://github.com/SuhasT11/aws-three-tier-web-architecture-workshop/blob/main/application-code/web-tier/src/assets/3TierArch.png)

In this architecture, a public-facing Application Load Balancer forwards client traffic to our web tier EC2 instances. The web tier is running Nginx webservers that are configured to serve a React.js website and redirects our API calls to the application tier’s internal facing load balancer. The internal facing load balancer then forwards that traffic to the application tier, which is written in Node.js. The application tier manipulates data in an Aurora MySQL multi-AZ database and returns it to our web tier. Load balancing, health checks and autoscaling groups are created at each layer to maintain the availability of this architecture.



iam role

	AmazonSSMManagedInstanceCore


vpc
	10.0.0.0/16


subnet
	public-web-subnet-az1
	10.0.0.0/24

	public-web-subnet-az2
	10.0.10.0/24

	private-app-subnet-az1
	10.0.20.0/24

	private-app-subnet-az2
	10.0.30.0/24

	private-db-subnet-az1
	10.0.40.0/24

	private-db-subnet-az2
	10.0.50.0/24


ig
	three-tier-ig


nat

	three-tier-nat-az1
	zonal
	public-subnet-az1


	three-tier-nat-az2
	zonal
	public-subnet-az2


route-table

	PublicRouteTable      web layer public subnets
	0.0.0.0   ig
	2 public subnet

	PrivateRouteTable-AZ-1
	0.0.0.0 nat-az1
	private-app-subnet-az1


	PrivateRouteTable-AZ-2
	0.0.0.0/0   nat-az2
	private-app-subnet-az2


sg
		internet-facing-lb-sg
		External load balancer security group
		http ->  80  ->  my IP

		WebTierSG
		SG for the web Tier
		http ->  80  ->  my IP
		http ->  80  -> internet-facing-lb-sg

		internal-lb-sg
		SG for the internal load balancer
		http ->  80  -> WebTierSG

		AppTierSG
		SG for the app Tier
		Custom TCP ->  4000  ->  my IP
		Custom TCP ->  4000 -> internal-lb-sg

		DBSG
		SG for database
		MYSQL->  3306 -> AppTierSG



DB

	DB subnet group

	three-tier-db-subnet-group
	Subnet group for the database layer
	az -2
	private-db-subnet-az1, 	private-db-subnet-az2


	Database creation

	Mysql 
	database-1
	admin
	MyPass123
	3-tier-vpc
	three-tier-db-subnet-group
	sg - DBSG
	create

	endpoint -> connectivity -> endpoint -> database-1.cros64qqu1iz.ap-south-1.rds.amazonaws.com



App Instance Deployment

	AppLayer
	aws linux kernel-6.18
	no key pair
	private-app-subnet-az1
	AppTierSG
	advanced details -> IAM instance profile -> ec2ssm

	connect using ssm

	sudo -su ec2-user
	ping 8.8.8.8

	sudo dnf install mariadb105 -y
	mysql --version
	mysql -h database-2.cros64qqu1iz.ap-south-1.rds.amazonaws.com -P 3306 -u admin -p 
	CREATE DATABASE webappdb;   
	SHOW DATABASES;
	USE webappdb;    
	CREATE TABLE IF NOT EXISTS transactions(id INT NOT NULL AUTO_INCREMENT, amount DECIMAL(10,2), description VARCHAR(100), PRIMARY KEY(id));  
	SHOW TABLES;    
	INSERT INTO transactions (amount,description) VALUES ('400','groceries');     
	SELECT * FROM transactions;





App code setup

	curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
	source ~/.bashrc
	nvm install 16
	nvm use 16
	npm install -g pm2   
	yum install git
	git clone https://github.com/SuhasT11/aws-three-tier-web-architecture-workshop.git
	cd aws-three-tier-web-architecture-workshop/application-code/app-tier
	update DbConfig.js
	    DB_HOST : 'database-2.cros64qqu1iz.ap-south-1.rds.amazonaws.com',
	    DB_USER : 'admin',
	    DB_PWD : 'MyPass123',
	    DB_DATABASE : 'webappdb'
    npm install
	pm2 t index.js
	pm2 list
	pm2 logs
	pm2 startup
	pm2 save
	curl http://localhost:4000/health
	curl http://localhost:4000/transaction



App Tier AMI

	AppTierImage
	App Tier

target group
	instances
	AppTierTargetGroup
	HTTP 4000
	vpc
	/health

	
Internal Load Balancer 
	app-tier-internal-lb
	internal
	vpc
	private app subnet - 2
	internal-lb-sg
	HTTP 80
	internal-lb-sg


Launch Template
	AppTierLaunchTemplate
	LaunchTemplate for AppTier
	select AppTierImage
	dont select subnet
	sg - AppTierSG
	IAM instance profile - ec2ssm


Auto Scaling group
	AppTierASG
	vpc
	private app subnet - 2
	lb - AppTierTargetGroup
	


Web Instance Deployment
	WebLayer
	aws linux kernel-6.18
	no key pair
	public-web-subnet-az1
	Auto-assign public IP - enabled
	WebTierSG
	advanced details -> IAM instance profile -> ec2ssm

	connect using ssm

	sudo -su ec2-user
	ping 8.8.8.8

Configure Web Instance
	curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
	source ~/.bashrc
	nvm install 16
	nvm use 16
	git clone https://github.com/SuhasT11/aws-three-tier-web-architecture-workshop.git
	cd aws-three-tier-web-architecture-workshop/application-code/
	Update Config File

		application-code/nginx.conf
		replace [INTERNAL-LOADBALANCER-DNS] with your internal load balancer’s DNS entry
	cd web-tier
	npm install 
	npm run build
	sudo dnf install nginx -y
	cd /etc/nginx
	ls
	sudo rm nginx.conf
	 cp /home/ec2-user/aws-three-tier-web-architecture-workshop/application-code/nginx.conf .
	 keep the build in this path /home/ec2-user/web-tier/build
	 sudo service nginx restart
	 chmod -R 755 /home/ec2-user
	 sudo chkconfig nginx on
	 verify using public IP

Web Tier AMI

	WebTierImage
	Image for web tier


Target groups
	WebTierTargetGroup
	HTTP 80
	vpc
	/health

Application Load Balancer
	web-tier-external-lb
	Internet-facing
	public subnet 2
	sg - WebTierSG
	WebTierTargetGroup

Create launch template
	WebTierLaunchTemplate
    Launch Template for WebTier
    AMI - WebTierImage
    t3micro
    sg - webTier-sg


Create Auto Scaling group
	WebTierASG
	WebTierLaunchTemplate
	VPC
	subnet public 2
	lb - WebTierTargetGroup
