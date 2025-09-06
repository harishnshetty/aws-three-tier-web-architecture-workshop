# AWS Three Tier Web Architecture Workshop
## For more projects, check out  
## [https://harishnshetty.github.io/projects.html](https://harishnshetty.github.io/projects.html)
## [youtube-Link](https://www.youtube.com/@devopsHarishNShetty)
## Architecture Overview
![Architecture Diagram](https://github.com/harishnshetty/image-data-project/blob/f8886e589538c1dc680d4c71baa450881cf7afcf/3tierworkshop.jpg)


---
```
1. VPC (6 Subnets, 5 Route Tables, 1 IGW, 1 NAT)
2. Security Group (Cross-connection)
3. EC2
4. Auto Scaling Group (ASG with IAM & Launch Template)
5. ALB (2 Target Groups)
6. IAM (for Image Build & Access Control)
7. RDS (MySQL)
6. Route 53
```


---

## Create a Security Group

| SG name      | inbound        | Access         | Description                                  |
|--------------|----------------|---------------|----------------------------------------------|
| Jump Server  | 22             | MY-ip         | access from my laptop                        |
| 1. web-frontend-alb     | 80,         | 0.0.0.0/24    | all access from internet                     |
| 2. Web-srv-sg      | 80,  22    | 1. web-frontend-alb       | only front-alb and jump server access        |
|              |                | jump-server   |                                              |
| 3. app-Internal-alb-sg     |  80,  | 2. Web-srv-sg      | only web-srv                                 |
| 4. app-Srv-sg      | 4000,  22 | 3. app-Internal-alb-sg | only 3. app-Internal-alb-sg and jump server access          |
|              |                | jump-server   |                                              |
| 5. DB-srv       | 3306, 22       | 4. app-Srv-sg       | only app-srv and jump server access          |
|              |      3306          | jump-server   |                                              |

---

## Create a VPC

| #  | Component         | Name                  | CIDR / Details                               |
|----|-------------------|-----------------------|----------------------------------------------|
| 1  | VPC              | 3-tier-vpc            | 10.75.0.0/16                                  |
| 12 | Subnets          | Public-Subnet-1a      | 10.75.1.0/24                                  |
|    |                  | Public-Subnet-1b      | 10.75.2.0/24                                  |
|    |                  | app-private-Subnet-1a | 10.75.3.0/24                                  |
|    |                  | app-private-Subnet-1b | 10.75.4.0/24                                  |
|    |                  | db-private-Subnet-1a  | 10.75.5.0/24                                  |
|    |                  | db-private-Subnet-1b  | 10.75.6.0/24                                  |

---
## Create a role for both web and app tier

- 3-tier-role:
    - AmazonS3ReadOnlyAccess
    - AmazonSSMManagedInstanceCore

---

## Create a Mysql in the RDS

### First Create the subnet Group

| Name    | three-subnet-gp-rds   |
|---------|-----------------------|
| VPC     | three-tier-rds-subnetgroup            |
| AZ      | 1a, b, c              |
| Subnets | DB-Private-Subnet-1a  |
|         | DB-Private-Subnet-1b  |


| Parameter                          | Value                |
|-------------------------------------|----------------------|
| DB instance identifier              | db-3tier             |
| Master username                     | admin                |
| Self managed password               | SuperadminPassword   |
| Instance class (Burstable)          | db.t3.small          |
| Storage                             | 20 GB                |
| Virtual private cloud VPC           | 3-tier-vpc           |
| Security Group (SG)                 | db-srv               |
| Enable Enhanced monitoring          | Unchecked            |
---




## Setup the Ec2-instance and create the IAM (WEB Tier)
**REF:** [web-tier](https://github.com/harishnshetty/3-tier-aws-15-services/edit/main/application-code/web-tier)

**Only Setup the Packages:**  
- Nginx  
- nvm  


```bash
sudo yum install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
sudo service nginx restart
sudo chkconfig nginx on
sudo yum install git -y

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

```

```bash
# Download and install nvm:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

# in lieu of restarting the shell
\. "$HOME/.nvm/nvm.sh"

# Download and install Node.js:
nvm install 22

# Verify the Node.js version:
node -v # Should print "v22.19.0".
nvm current # Should print "v22.19.0".

# Verify npm version:
npm -v # Should print "10.9.3".
```

```bash
git clone https://github.com/aws-samples/aws-three-tier-web-architecture-workshop.git
```


---

## Setup the Ec2-instance and create the IAM (APP Tier)
**REF:** [app-tier](https://github.com/harishnshetty/3-tier-aws-15-services/tree/main/application-code/app-tier)

**Only Setup the Packages:**  
- install  
    - mysql client  
    - nvm  
    - pm2  



```bash
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install mysql80-community-release-el9-1.noarch.rpm -y
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf install mysql-community-client -y
mysql --version
```
```bash
mysql -h CHANGE-TO-YOUR-RDS-ENDPOINT -u CHANGE-TO-USER-NAME -p
```

```sql
CREATE DATABASE webappdb;
SHOW DATABASES;
USE webappdb;
CREATE TABLE IF NOT EXISTS transactions(id INT NOT NULL AUTO_INCREMENT, amount DECIMAL(10,2), description VARCHAR(100), PRIMARY KEY(id));
SHOW TABLES;
INSERT INTO transactions (amount,description) VALUES ('400','groceries');
SELECT * FROM transactions;
```

```bash 
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

```

```bash
# Download and install nvm:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

# in lieu of restarting the shell
\. "$HOME/.nvm/nvm.sh"

# Download and install Node.js:
nvm install 22
npm install -g pm2

# Verify the Node.js version:
node -v # Should print "v22.19.0".
nvm current # Should print "v22.19.0".

# Verify npm version:
npm -v # Should print "10.9.3".
```

```bash
git clone https://github.com/aws-samples/aws-three-tier-web-architecture-workshop.git
```
---

## Create images for both web and app Tier
- Web-Tier-IAM-IMAGE  
- APP-Tier-IAM-IMAGE  

---

## Create the  S3 Buckets

```bash
git clone https://github.com/harishnshetty/3-tier-aws-15-services.git
```

1. 3-tier-aws-project-8745 (upload your content)  
2. 3tier-vpc-flow-log-8745 (attach this bucket this immediately) with arn Value [ arn:aws:s3:::3tier-vpc-flow-log-8745 ]

---



## Now add the Database into the RDS-MYSQL

**Ref:** https://catalog.us-east-1.prod.workshops.aws/workshops/85cd2bb2-7f79-4e96-bdee-8078e469752a/en-US/part3/configuredatabase


---



## Create web launch template

| Parameter              | Value                |
|------------------------|----------------------|
| Name                   | web-tier-lt          |
| My AMI's               | Web-Tier-IAM-IMAGE   |
| Security Groups        | Web-srv-sg           |
| IAM Instance Profile   | 3-tier-web-role      |

**User Data:**
```bash
#!/bin/bash
# Log everything to /var/log/user-data.log
exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1

# Install AWS CLI v2 (if not already)
yum install -y awscli

# Download application code from S3
aws s3 cp s3://<YOUR-S3-BUCKET-NAME>/application-code /home/ec2-user/application-code --recursive

# Go to app directory
cd /home/ec2-user/application-code

# Make script executable and run it
chmod +x web.sh
sudo ./web.sh
```

---

## Create app launch template

| Parameter              | Value                |
|------------------------|----------------------|
| Name                   | app-tier-lt          |
| My AMI's               | app-Tier-IAM-IMAGE   |
| Security Groups        | app-Srv-sg           |
| IAM Instance Profile   | 3-tier-web-role      |

**User Data:**
```bash
#!/bin/bash
# Log everything to /var/log/user-data.log
exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1

# Install AWS CLI v2 (if not already)
yum install -y awscli

# Download application code from S3
aws s3 cp s3://<YOUR-S3-BUCKET-NAME>/application-code /home/ec2-user/application-code --recursive

# Go to app directory
cd /home/ec2-user/application-code

# Make script executable and run it
chmod +x app.sh
sudo ./app.sh
```

---

## Create target group 

| Tier      | Name      | Port  | VPC         | Health-check   |
|-----------|-----------|-------|-------------|---------------|
| Web Tier  | Web-tier  | 80    | 3-tier-vpc  |               |
| App Tier  | App-tier  | 4000  | 3-tier-vpc  | /health        |
---

## Create Load balancers

### Application Load Balancers
| Load Balancer | Name     | Type            | VPC        | Availability Zones                                 | Security Groups | Listeners & Routing   |
|---------------|----------|-----------------|------------|---------------------------------------------------|-----------------|----------------------|
| app-alb       | app-alb  | Internal-facing | 3-tier-vpc | App-Private-Subnet-1a, 1b, 1c                     | app-Internal-alb-sg         | 80 app-tier          |
| web-alb       | web-alb  | Internet-facing | 3-tier-vpc | Public-Subnet-1a, 1b, 1                           | web-frontend-alb        | 80 web-tier          |
---

## Immediately update the `nginx.config` of your internal load balancer Address
---
## Create Auto Scaling

| Name            | Launch template | Instance types | VPC        | Subnets (AZs)                       | Load balancer | Desired | min | max | Scaling policy | Notifications    | Tag      |
|-----------------|----------------|---------------|------------|--------------------------------------|---------------|---------|-----|-----|---------------|-----------------|----------|
| web-tier-asg    | web-tier-lb    | t2.micro      | 3-tier-vpc | Web-Private-Subnet-1a, 1b, 1c        | web-tier      | 3       | 3   | 6   | 60            | web-tier-sns    | web-asg  |
| app-tier-asg    | app-tier-lb    | t2.micro      | 3-tier-vpc | app-Private-Subnet-1a, 1b, 1c        | app-tier      | 3       | 3   | 6   | 60            | app-tier-sns    | app-asg  |

---

## Configure the Route53  
