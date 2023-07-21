# Project-4: Re-Architecting Web App on AWS Cloud[Cloud Native]

[*Project Source*](https://www.udemy.com/course/decodingdevops/learn/lecture/26464908#overview)


![Architecture](Images/Architecture%20setup.png)


## Pre-requisites:
  * AWS Account
  * Default VPC
  * Route53 Public Registered Name
  * Maven
  * JDK8

  
### Step-1: Create Keypair for Beanstalk EC2 Login

- Create a key pair to be used with Elastic Beanstalk. Go to `EC2` console, select `KeyPair` -> `Create key pair`.
```sh
Name: vprofile-bean-key
```
- Remember where you download the private key, it will be used when logging in to EC2 via SSH.

### Step-2: Create Security Group for ElastiCache, RDS and ActiveMQ

- Create a Security Group with name `vprofile-backend-SG`. Once it is created we edit `Inbound` rules:
```sh
All Traffic from `vprofile-backend-SG`
```  

### Step-3: Create RDS Database

#### Create Subnet Group:

- Create `Subnet Groups` with below properties:
```sh
Name: vprofile-rds-sub-grp
AZ: Select All
Subnet: Select All
```

#### Create Parameter Group

- Create a parameter group to be used with our RDS instance. 
- With parameter group, we are able make updates to default parameter for our RDS instance.

```sh
Parameter group family: mysql8.0
Type: DB Parameter Group
Group Name: vprofile-rds-para-grp
```

![](Images/Parameter%20group.png)

#### Create Database

- Create RDS instance with below properties:
```sh
Method: Standard Create
Engine Options: MySQL
Engine version: 8.0.32
Templates: Free-Tier
DB Instance Identifier: vprofile-rds-mysql
Master username: admin
Password: Auto generate psw
Instance Type: db.t2.micro
Subnet grp: vprofile-rds-sub-grp
SecGrp:  vprofile-backend-SG
No public access
DB Authentication: Password authentication
Additional Configuration
Initial DB Name: accounts
Keep the rest default or you may add as your own preference
```

- After clicking `Create` button, you will see a popup. Click `View credential details` and note down auto-generated db password. It will be used in the application config files.

### Step-4: Create ElastiCache


#### Create Subnet Group:

- Create `Subnet Groups` with below properties:
```sh
Name: vprofile-memcached-sub-grp
AZ: Select All
Subnet: Select All
```

#### Create Parameter Group

- Create a parameter group to be used with the ElastiCache instance. 
```sh
Name: vprofile-memcached-para-grp
Description: vprofile-memcached-para-grp
Family: memcached1.4
```


#### Create Memcached Cluster

- Navigate to `Get Started` -> `Create Clusters` -> `Memcached Clusters`
```sh
Name: vprofile-elasticache-svc
Engine version: 1.6.17
Parameter Grp: vprofile-memcached-para-grp
NodeType: cache.t2.micro
# of Nodes: 1
SecGrp: vprofile-backend-SG
```

### Step-5: Create Amazon MQ

- We will create Amazon MQ service with below properties:
```sh
Engine type: RabbitMQ
Single-instance-broker
Broker name: vprofile-rmq
Instance type: mq.t3.micro
username: rabbit
psw: bunnyhole789
Additional Settings:
private Access
VPC: use default
SEcGrp: vprofile-backend-SG
```

- Always note down your username/pwd as you won't be able to retrieve the Password from console.

### Step-6: DB Initialization

- Navigate to RDS instance copy endpoint.
```sh
vprofile-rds-mysql.cjleey4jxpio.us-east-1.rds.amazonaws.com
```

- Create an EC2 instance to initialize the DB, this instance will be terminated after initialization.
```sh
Name: mysql-client
OS: ubuntu 22.04
t2.micro
SecGrp: Allow SSH on port 22
Keypair: vprofile-prod-key
Userdata:
#! /bin/bash
apt update -y
apt upgrade -y
apt install mysql-client -y
```

### Step-7: Update Backend Security Group & ELB
- The application instances created by BeanStalk need to communicate with Backend services.
- Update `vprofile-backend-SG` Inbound rule to allow connection from `mysql-client-SG` on port `3306` 

```sh
Custom TCP 3306 from Instance mysql-client-SG
```

- SSH into `mysql-client` instance, then update and install mysql-client

```sh
ssh -i Downloads/vprofile-bean-key.pem ubuntu@<public IP address> given
sudo apt update && sudo apt install mysql-client -y
```

- After updating the Inbound rule `vprofile-backend-SG` connect with command below:
```sh
mysql -h vprofile-rds-mysql.cjleey4jxpio.us-east-1.rds.amazonaws.com -u admin -p<db_password>accounts
```
![](Images/mysql%20connection.png)

- Next, clone the source code here to use script to initialize our database. After these commands we should be able to see 2 tables `role`, `user`, and `user_role`.

```sh
git clone https://github.com/devopshydclub/vprofile-project.git
ls
cd vprofile-project/
ls
ls src/main/resources/db_backup.sql
mysql -h vprofile-rds-mysql.cjleey4jxpio.us-east-1.rds.amazonaws.com -u admin -p<db_password> accounts <src/main/resources/db_backup.sql
mysql -h vprofile-rds-mysql.cjleey4jxpio.us-east-1.rds.amazonaws.com -u admin -p<db_password> accounts
show tables;
```

### Step-8: Create Elastic Beanstalk Environment

- The backend services are ready now. 
- Copy their endpoints from AWS console. These information will be used in our `application.properties` file
```sh
RDS:
vprofile-rds-mysql.chrgxmhxkprk.us-east-1.rds.amazonaws.com:3306
ActiveMQ: amqps://b-b7d7bbcb-3894-4af7-8048-726a9ceabc43.mq.us-east-1.amazonaws.com:5671
ElastiCache:
vprofile-elasticache-svc.eqmmsw.cfg.use1.cache.amazonaws.com:11211
```

#### Create Application

- Create a role for Beanstalk and give it permission to access Backend services. vprofile-bean-role
- Application in Elastic Beanstalk can accommodate  multiple environments. Since the app is Running on Tomcat we will choose `Tomcat` as platform.
```sh
Name: vprofile-app
Environment: Vprofileapp-prod
Domain: vprofileapp-ym
Platform: Tomcat
keep the rest default
Configure more options:
- Custom configuration
****Instances****
EC2 SecGrp: vprofile-backend-SG
****Capacity****
LoadBalanced
Min:2
Max:8
InstanceType: t2.micro
****Rolling updates and deployments****
Deployment policy: Rolling
Percentage :50 %
****Security****
EC2 key pair: vprofile-bean-key
```
 PS:  Domain needs to be unique & check the availability after choosing it.

 ![](Images/Environment%20created.png)


### Step-9: Update Backend SecGrp & ELB

- In Elastic Beanstalk console, under the app environment, click Configuration and apply the  below changes:
Enable Access control list on s3 bucket.
```sh
Add Listener HTTPS port 443 with SSL cert
Processes: Health check path : /login
```

- Update vprofile-backend-SG to allow connection from our appSecGrp created by Beanstalk on port 3306, 11211 and 5671
```sh
Custom TCP 3306 from Beanstalk SecGrp(you can find id from EC2 insatnces)
Custom TCP 11211 from Beanstalk SecGrp
Custom TCP 5671 from Beanstalk SecGrp
```
![](Images/Backend%20security%20group.png)


### Step-10: Build and Deploy Artifact

- Navigate the project directory , checkout aws-refactor branch.
- Update `application.properties` file with correct endpoints, username and password.
```sh
vim src/main/resources/application.properties
*****Updates*****
jdbc.url
jdbc.password
memcached.active.host
rabbitmq.address
rabbitmq.username
rabbitmq.password
```

- Navigate the project root directory to  `pom.xml` file. Run below command to build the artifact.
```sh
mvn install
``` 

#### Upload Artifact to Elastic Beanstalk

- Go to Application versions and Upload the artifact from your local. It will autmatically upload the artifact to the S3 bucket created by Elasticbeanstalk.

- Select the uploaded application `vprofile-v2.war` and click Deploy.

![](Images/App-%20deployed.png)

-  application deployed successfully.

![](Images/App%20deployed%20from%20Beanstalk.png)


### Step-11: Create DNS Record in Route53 for Application

- Create an A record which aliasing Elastic Beanstalk endpoint.

- Now we can reach our application securely with DNS name we have given.


### Step-12: Create Cloudfront Distribution for CDN

- Cloudfront is Content Delivery Nettwork service of AWS. It uses Edge Locations around the world to deliver contents globally with best performance.  `CloudFront` is used to  create network distribution.
```sh
Origin Domain: DNS record name we created for our app in previous step
Viewer protocol: Redirect HTTP to HTTPS
Alternate domain name: DNS record name we created for our app in previous step
SSL Certificate: 
Security policy: TLSv1
``` 
- Now we can check our application from browser.

![](Images/App%20running%20from%20CDN.png)

### Step-9: Clean-up

- Delete all resources created during the project.