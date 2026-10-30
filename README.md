# Deploy a Three-Tier Web Application on AWS

> A highly available, highly secured, highly scalable, and fault-tolerant three-tier web application deployed on Amazon Web Services.

**Author:** Subodh Kumar  
**GitHub:** [github.com/SubodhK143](https://github.com/SubodhK143)  
**LinkedIn:** [linkedin.com/in/subodh-kumar-aws-certified](https://www.linkedin.com/in/subodh-kumar-aws-certified/)  
**Source Code:** [Three_Tire_App_On_AWS](https://github.com/Subodhk143/Three_Tier_App_On_AWS.git)

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [AWS Services Used](#aws-services-used)
- [Network Design](#network-design)
- [Security Groups](#security-groups)
- [Implementation Steps](#implementation-steps)
  - [1. Create VPC and Subnets](#1-create-vpc-and-subnets)
  - [2. Create Internet Gateway, NAT Gateway, and Route Tables](#2-create-internet-gateway-nat-gateway-and-route-tables)
  - [3. Create Security Groups](#3-create-security-groups)
  - [4. Create Database Subnet Group and RDS](#4-create-database-subnet-group-and-rds)
  - [5. Create IAM Role](#5-create-iam-role)
  - [6. Create S3 Bucket and Upload Source Code](#6-create-s3-bucket-and-upload-source-code)
  - [7. Create and Configure Test App Server](#7-create-and-configure-test-app-server)
  - [8. Create AMI of App Server](#8-create-ami-of-app-server)
  - [9. App Tier — Target Group, Launch Template, Internal Load Balancer](#9-app-tier--target-group-launch-template-internal-load-balancer)
  - [10. Create and Configure Test Web Server](#10-create-and-configure-test-web-server)
  - [11. Create AMI of Web Server](#11-create-ami-of-web-server)
  - [12. Web Tier — Target Group, Launch Template, External Load Balancer](#12-web-tier--target-group-launch-template-external-load-balancer)
  - [13. Create Auto Scaling Groups](#13-create-auto-scaling-groups)
  - [14. Access the Application](#14-access-the-application)
  - [15. Cleanup](#15-cleanup)
- [Notes](#notes)

---

## Architecture Overview

The three-tier architecture separates the application into three distinct layers:

| Tier | Layer | Description |
|------|-------|-------------|
| **Web Tier** | Presentation (Frontend) | User interface — presents information and collects user input |
| **App Tier** | Application (Backend) | Business logic — processes requests, performs computations, and makes decisions |
| **DB Tier** | Database | Data storage — stores, retrieves, and manages application data |

**Request flow:**
```
User → Web Tier → App Tier → Database Tier → App Tier → Web Tier → User
```

Resources are distributed across **two Availability Zones** (ap-south-1a and ap-south-1b) within a custom VPC for high availability and fault tolerance.

---

## AWS Services Used

| # | Service | Purpose |
|---|---------|---------|
| 1 | Amazon EC2 | Virtual servers for web and app tiers |
| 2 | Amazon S3 | Source code storage |
| 3 | Amazon VPC | Isolated network environment |
| 4 | Auto Scaling Group (ASG) | Automatic horizontal scaling |
| 5 | Application Load Balancer (ALB) | Traffic distribution (external & internal) |
| 6 | IAM | Secure access management |
| 7 | Amazon RDS (Aurora MySQL) | Managed relational database |
| 8 | Amazon Machine Image (AMI) | Server snapshots for launch templates |
| 9 | Launch Templates | Automated EC2 instance configuration |
| 10 | Target Groups | ALB routing targets |
| 11 | Internet Gateway | Public internet access for web tier |
| 12 | NAT Gateway | Outbound internet for private subnets |
| 13 | Route Tables | Network traffic routing |
| 14 | Security Groups | Firewall rules per tier |

---

## Network Design

**VPC CIDR:** `10.0.0.0/16`

| Subnet Name | Type | Availability Zone | CIDR Block |
|-------------|------|-------------------|------------|
| Web-Tier-Subnet-1-Public | Public | ap-south-1a | 10.0.1.0/24 |
| App-Tier-Subnet-1-Private | Private | ap-south-1a | 10.0.2.0/24 |
| DB-Tier-Subnet-1-Private | Private | ap-south-1a | 10.0.3.0/24 |
| Web-Tier-Subnet-2-Public | Public | ap-south-1b | 10.0.4.0/24 |
| App-Tier-Subnet-2-Private | Private | ap-south-1b | 10.0.5.0/24 |
| DB-Tier-Subnet-2-Private | Private | ap-south-1b | 10.0.6.0/24 |

> **Design principle:** The Web Tier uses public subnets (internet-facing). The App and DB tiers use private subnets (internal only) for security.

**Route Tables:**

| Route Table | Subnet Association | Target |
|------------|-------------------|--------|
| Web-Tier-RT | Web-Tier-Subnet-1-Public, Web-Tier-Subnet-2-Public | Internet Gateway |
| App-Tier-RT-1 | App-Tier-Subnet-1-Private | NAT-GW-1 |
| App-Tier-RT-2 | App-Tier-Subnet-2-Private | NAT-GW-2 |

---

## Security Groups

| Security Group | Protocol / Port | Source |
|----------------|----------------|--------|
| External-Load-Balancer-SG | HTTP (80) | 0.0.0.0/0 |
| Web-Tier-SG | HTTP (80) | External-Load-Balancer-SG |
| Internal-Load-Balancer-SG | HTTP (80) | Web-Tier-SG |
| App-Tier-SG | Custom TCP (4000) | Internal-Load-Balancer-SG |
| DB-Tier-SG | MySQL (3306) | App-Tier-SG |

---

## Implementation Steps

### 1. Create VPC and Subnets

1. Navigate to **VPC > Your VPCs > Create VPC**
2. Set **CIDR block** to `10.0.0.0/16`, name it `Three-tier-project-VPC`
3. Navigate to **VPC > Subnets > Create Subnet** and create all 6 subnets per the table above
4. For both public web subnets, go to **Actions > Edit Subnet Settings** and enable **Auto-assign public IPv4 address**

---

### 2. Create Internet Gateway, NAT Gateway, and Route Tables

**Internet Gateway**
1. Navigate to **VPC > Internet Gateways > Create Internet Gateway**
2. After creation, go to **Actions > Attach to VPC** and select your VPC

**NAT Gateways**
1. Navigate to **VPC > NAT Gateways > Create NAT Gateway**
2. Create `NAT-GW-1` in `Web-Tier-Subnet-1-Public` with a new Elastic IP
3. Create `NAT-GW-2` in `Web-Tier-Subnet-2-Public` with a new Elastic IP

**Route Tables**
1. Create `Web-Tier-RT`, `App-Tier-RT-1`, and `App-Tier-RT-2` in your VPC
2. For each, go to **Actions > Edit Routes** and add:
   - `Web-Tier-RT` → Destination `0.0.0.0/0` → Target: Internet Gateway
   - `App-Tier-RT-1` → Destination `0.0.0.0/0` → Target: NAT-GW-1
   - `App-Tier-RT-2` → Destination `0.0.0.0/0` → Target: NAT-GW-2
3. Go to **Actions > Edit Subnet Associations** and associate the appropriate subnets

---

### 3. Create Security Groups

Navigate to **VPC > Security Groups > Create Security Group** and create all 5 security groups using the inbound rules defined in the [Security Groups](#security-groups) table above.

---

### 4. Create Database Subnet Group and RDS

**DB Subnet Group**
1. Navigate to **RDS > Subnet Groups > Create DB Subnet Group**
2. Name: `DB-Subnet-Group-Three-Tier-Application`
3. Select VPC, choose Availability Zones `ap-south-1a` and `ap-south-1b`
4. Add DB subnets: `10.0.3.0/24` and `10.0.6.0/24`

**RDS Instance**
1. Navigate to **RDS > Databases > Create Database**
2. Engine: **Aurora (MySQL Compatible)**, Template: **Production**
3. DB Cluster Identifier: `RDS-Three-Tier`
4. Set a master username and password — **note these down for later**
5. Instance class: `db.t3.medium`, VPC: your custom VPC
6. DB Subnet Group: the one created above
7. Public Access: **No**, Security Group: `DB-Tier-SG`
8. Click **Create Database** (creation takes several minutes)

---

### 5. Create IAM Role

1. Navigate to **IAM > Roles > Create Role**
2. Trusted entity: **AWS Service**, Use case: **EC2**
3. Attach the following permission policies:
   - `AmazonS3ReadOnlyAccess`
   - `AmazonSSMManagedInstanceCore`
4. Name the role: `IAM-role-three-tier-app`

> This role allows EC2 instances to access the S3 bucket and connect via Session Manager — without requiring SSH keys or a bastion host.

---

### 6. Create S3 Bucket and Upload Source Code

1. Navigate to **S3 > Create Bucket** — choose a globally unique bucket name
2. Clone the source code repository:
   ```bash
   git clone https://github.com/Satyanarayan4434/Three_Tire_App_On_AWS.git
   ```
3. Upload the `application-code/` folder to the S3 bucket

**Update `DbConfig.js`** before uploading:
```js
module.exports = Object.freeze({
  DB_HOST: "<RDS Writer Instance Endpoint>",
  DB_USER: "admin",
  DB_PWD: "<your-password>",
  DB_DATABASE: "webappdb",
});
```

Replace the existing `DbConfig.js` in `s3://<bucket>/application-code/app-tier/` with the updated file.

---

### 7. Create and Configure Test App Server

**Launch EC2 Instance**
1. Navigate to **EC2 > Instances > Launch Instances**
2. Name: `Test-APP-Server`, AMI: `Amazon Linux 2023`
3. Instance type: `t2.micro`
4. VPC: your custom VPC, Subnet: `App-Tier-Subnet-1-Private`
5. Security Group: `App-Tier-SG`
6. IAM Instance Profile: `IAM-role-three-tier-app`

**Connect via Session Manager** (no SSH key required):
```
EC2 > Instances > Test-APP-Server > Connect > Session Manager > Connect
```

**Configure the App Server:**
```bash
# Switch to ec2-user
sudo -su ec2-user

# Install MySQL client
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install mysql80-community-release-el9-1.noarch.rpm -y
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf install mysql-community-client -y

# Connect to RDS
mysql -h <RDS-Endpoint> -u admin -p

# Set up the database
CREATE DATABASE webappdb;
USE webappdb;
CREATE TABLE IF NOT EXISTS transactions(
  id INT NOT NULL AUTO_INCREMENT,
  amount DECIMAL(10,2),
  description VARCHAR(100),
  PRIMARY KEY(id)
);
INSERT INTO transactions (amount, description) VALUES ('400', 'groceries');
exit

# Copy app code from S3
cd /home/ec2-user
sudo aws s3 cp s3://<YOUR-S3-BUCKET-NAME>/application-code/app-tier app-tier --recursive
cd app-tier
sudo chown -R ec2-user:ec2-user /home/ec2-user/app-tier
sudo chmod -R 755 /home/ec2-user/app-tier

# Install Node.js and dependencies
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 16
nvm use 16
npm install -g pm2
npm install
npm audit fix

# Start the app and configure pm2 auto-start
pm2 start index.js
pm2 startup
# Run the sudo env command output by pm2 startup
pm2 save

# Verify health
curl http://localhost:4000/health
```

---

### 8. Create AMI of App Server

1. Go to **EC2 > Instances**, select `Test-APP-Server`
2. Navigate to **Actions > Image and Templates > Create Image**
3. Name: `APP-server-image`
4. Click **Create Image** and wait for the AMI to become available

---

### 9. App Tier — Target Group, Launch Template, Internal Load Balancer

**Target Group**
- Navigate to **EC2 > Target Groups > Create Target Group**
- Protocol: `HTTP`, Port: `4000`, VPC: your custom VPC
- Health check path: `/health`
- Name: `App-Server-TG`

**Launch Template**
- Navigate to **EC2 > Launch Templates > Create Launch Template**
- Name: `App-Server-Launch-template`
- AMI: `APP-server-image` (from My AMIs)
- Instance type: `t2.micro`, Security Group: `App-Tier-SG`
- IAM Instance Profile: `IAM-role-three-tier-app`

**Internal Application Load Balancer**
- Navigate to **EC2 > Load Balancers > Create Load Balancer**
- Name: `Application-load-balancer`, Scheme: **Internal**
- VPC: your custom VPC, Subnets: `App-Tier-Subnet-1-Private` and `App-Tier-Subnet-2-Private`
- Security Group: `Internal-Load-Balancer-SG`
- Listener: `HTTP:80` forwarding to `App-Server-TG`

---

### 10. Create and Configure Test Web Server

**Update `nginx.conf`** with the internal load balancer DNS name:
```nginx
location /api/ {
    proxy_pass http://<Internal-Load-Balancer-DNS-Name>:80/;
}
```

Upload the updated `nginx.conf` to your S3 bucket (replace the existing file).

**Launch EC2 Instance**
- Name: `Test-Web-Server`, Subnet: `Web-Tier-Subnet-1-Public`
- Security Group: `Web-Tier-SG`, IAM Profile: `IAM-role-three-tier-app`

**Configure the Web Server:**
```bash
sudo -su ec2-user
cd /home/ec2-user

# Copy web code from S3
sudo aws s3 cp s3://<S3_BUCKET_NAME>/application-code/web-tier web-tier --recursive
sudo chown -R ec2-user:ec2-user /home/ec2-user
sudo chmod -R 755 /home/ec2-user

# Install Node.js and build the React app
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 16 && nvm use 16
cd /home/ec2-user/web-tier
npm install
npm run build

# Install and configure Nginx
sudo yum install nginx -y
cd /etc/nginx
sudo mv nginx.conf nginx-backup.conf
sudo aws s3 cp s3://<S3_BUCKET_NAME>/application-code/nginx.conf .
sudo chmod -R 755 /home/ec2-user
sudo service nginx restart
sudo chkconfig nginx on
```

---

### 11. Create AMI of Web Server

1. Select `Test-Web-Server` in EC2 Instances
2. Navigate to **Actions > Image and Templates > Create Image**
3. Name: `Web-Server-Image`

---

### 12. Web Tier — Target Group, Launch Template, External Load Balancer

**Target Group**
- Protocol: `HTTP`, Port: `80`, VPC: your custom VPC
- Name: `Web-Server-TG`

**Launch Template**
- AMI: `Web-Server-Image`, Security Group: `Web-Tier-SG`
- IAM Instance Profile: `IAM-role-three-tier-app`

**External (Internet-Facing) Application Load Balancer**
- Scheme: **Internet-facing**
- Subnets: `Web-Tier-Subnet-1-Public` and `Web-Tier-Subnet-2-Public`
- Security Group: `External-Load-Balancer-SG`
- Listener: `HTTP:80` forwarding to `Web-Server-TG`

---

### 13. Create Auto Scaling Groups

**App Tier ASG**
1. Navigate to **EC2 > Auto Scaling Groups > Create Auto Scaling Group**
2. Name: `App-Tier-ASG`, Launch Template: `App-Server-Launch-template`
3. VPC: your custom VPC, Subnets: `App-Tier-Subnet-1-Private`, `App-Tier-Subnet-2-Private`
4. Attach to existing load balancer target group: `App-Server-TG`
5. Desired: `2`, Min: `2`, Max: `2`

**Web Tier ASG**
- Same process, using Web Tier components
- Subnets: `Web-Tier-Subnet-1-Public`, `Web-Tier-Subnet-2-Public`
- Target group: `Web-Server-TG`

> After both ASGs are running and healthy, you can terminate the two test instances (`Test-APP-Server`, `Test-Web-Server`).

---

### 14. Access the Application

1. Navigate to **EC2 > Load Balancers**
2. Select the **External Load Balancer** and copy its **DNS name**
3. Paste the DNS name into your browser

The application will display the **AWS 3-Tier Web App Demo**. Navigate to the **DB DEMO** section to verify the database connection is working and that sample data is visible.

---

### 15. Cleanup

Delete resources in the following order to avoid dependency errors:

| # | Resource |
|---|----------|
| 1 | Auto Scaling Groups |
| 2 | ALB Listeners, then Load Balancers |
| 3 | Target Groups |
| 4 | Launch Templates |
| 5 | AMIs |
| 6 | EBS Snapshots |
| 7 | EC2 Instances |
| 8 | Aurora Database *(disable delete protection first; skip final snapshot)* |
| 9 | NAT Gateways |
| 10 | Elastic IPs |
| 11 | Route Tables *(remove all subnet associations first)* |
| 12 | Internet Gateways *(detach from VPC first)* |
| 13 | VPC |
| 14 | S3 Bucket |
| 15 | IAM Role |

---

## Notes

- **Custom domain:** The application is accessed via the External Load Balancer's DNS name by default. For a user-friendly URL, purchase a domain and configure it using **Amazon Route 53**.
- **Cost awareness:** RDS Aurora and NAT Gateways incur ongoing charges. Delete the project after use to avoid unexpected costs.
- **Region:** This guide uses the **Mumbai (ap-south-1)** region. Any AWS region can be used — just ensure consistent Availability Zone selections throughout.
- **Instance naming:** All component names used in this guide are suggestions. You may use any names you prefer, as long as you reference them consistently.

---

*Documentation by Subodh Kumar — [GitHub](https://github.com/SubodhK143) | [LinkedIn](https://www.linkedin.com/in/subodh-kumar-aws-certified/)*
