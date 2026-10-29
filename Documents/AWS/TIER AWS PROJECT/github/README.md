# Three-Tier Web Application on AWS

A highly available, secure, scalable and fault-tolerant three-tier web application deployed on AWS (Mumbai region, `ap-south-1`) across two Availability Zones.

> Source code: https://github.com/SubodhK143/three-tier-app-on-aws
> Full step-by-step guide with screenshots: [`docs/three-tier-aws-guide.pdf`](docs/three-tier-aws-guide.pdf)

## Architecture

![Architecture](docs/images/architecture.png)

| Tier | Purpose | Subnet type | Tech |
|------|---------|-------------|------|
| Web (presentation) | React frontend served by Nginx, reverse-proxies `/api/` to the internal load balancer | Public | EC2 + Nginx + Auto Scaling Group |
| App (logic) | Node.js backend on port 4000 managed by PM2 | Private | EC2 + Auto Scaling Group |
| Database | MySQL-compatible Aurora, Multi-AZ (writer + reader) | Private | Amazon Aurora (MySQL) |

Traffic flow: `Internet -> Internet Gateway -> External ALB -> Web tier -> Internal ALB -> App tier -> Aurora`

## AWS services used

EC2, S3, VPC, Auto Scaling Groups, Application Load Balancers (internal and external), IAM, RDS/Aurora, AMI, Launch Templates, Target Groups, Internet Gateway, NAT Gateway, Route Tables, Security Groups.

## Network layout (VPC `10.0.0.0/16`)

| Subnet | AZ | CIDR | Type |
|--------|----|------|------|
| Web-Tier-Subnet-1-Public | ap-south-1a | 10.0.1.0/24 | Public |
| App-Tier-Subnet-1-Private | ap-south-1a | 10.0.2.0/24 | Private |
| DB-Tier-Subnet-1-Private | ap-south-1a | 10.0.3.0/24 | Private |
| Web-Tier-Subnet-2-Public | ap-south-1b | 10.0.4.0/24 | Public |
| App-Tier-Subnet-2-Private | ap-south-1b | 10.0.5.0/24 | Private |
| DB-Tier-Subnet-2-Private | ap-south-1b | 10.0.6.0/24 | Private |

Route tables:

- `Web-Tier-RT`: `0.0.0.0/0 -> Internet Gateway` (both public subnets)
- `App-Tier-RT-1`: `0.0.0.0/0 -> NAT-GW-1` (App subnet 1)
- `App-Tier-RT-2`: `0.0.0.0/0 -> NAT-GW-2` (App subnet 2)

## Security groups

| Security group | Inbound port | Source |
|----------------|--------------|--------|
| External-Load-Balancer-SG | HTTP 80 | 0.0.0.0/0 |
| Web-Tier-SG | HTTP 80 | External-Load-Balancer-SG |
| Internal-Load-Balancer-SG | HTTP 80 | Web-Tier-SG |
| App-Tier-SG | Custom TCP 4000 | Internal-Load-Balancer-SG |
| DB-Tier-SG | MySQL 3306 | App-Tier-SG |

## Deployment steps (summary)

1. **VPC and subnets**: create the VPC and the six subnets; enable auto-assign public IPv4 on the two web subnets.
2. **Gateways and routing**: Internet Gateway (attach to VPC), two NAT Gateways (one per public subnet), three route tables with the associations above.
3. **Security groups**: create the five groups from the table above.
4. **Database**: create a DB subnet group (DB subnets in 1a and 1b), then an Aurora (MySQL compatible) cluster with a writer and a reader, attached to `DB-Tier-SG`.
5. **IAM role**: EC2 role with `AmazonS3ReadOnlyAccess` and `AmazonSSMManagedInstanceCore`.
6. **S3 bucket**: create a private bucket and upload the `application-code` folder.
7. **App tier test server**: launch an EC2 instance in a private app subnet, connect via SSM Session Manager, set up the DB and the Node.js app (commands below), then create an AMI.
8. **App tier scaling**: target group (port 4000, health check `/health`), launch template, internal ALB, Auto Scaling Group.
9. **Web tier test server**: launch an EC2 instance in a public web subnet, build the React app, configure Nginx, then create an AMI.
10. **Web tier scaling**: target group (port 80, health check `/health`), launch template, internet-facing ALB, Auto Scaling Group.
11. **Access**: open the external load balancer DNS name in a browser.

## Configuration before upload to S3

Edit these files in `application-code/` and **do not commit real values**:

**`app-tier/DbConfig.js`**

```js
module.exports = Object.freeze({
  DB_HOST: "<aurora-writer-endpoint>",
  DB_USER: "<db-username>",
  DB_PWD: "<db-password>",
  DB_DATABASE: "webappdb",
});
```

**`nginx.conf`**: set the `/api/` proxy target:

```nginx
location /api/ {
    proxy_pass http://<internal-alb-dns-name>:80/;
}
```

## Key commands

**Install MySQL client and create the database (on the app test server)**

```bash
sudo -su ec2-user
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install mysql80-community-release-el9-1.noarch.rpm -y
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf install mysql-community-client -y

mysql -h <writer-endpoint> -u <username> -p
```

```sql
CREATE DATABASE webappdb;
USE webappdb;
CREATE TABLE IF NOT EXISTS transactions(
  id INT NOT NULL AUTO_INCREMENT,
  amount DECIMAL(10,2),
  description VARCHAR(100),
  PRIMARY KEY(id)
);
INSERT INTO transactions (amount, description) VALUES ('400', 'groceries');
```

**App tier**

```bash
cd /home/ec2-user
sudo aws s3 cp s3://<your-bucket>/application-code/app-tier app-tier --recursive
cd app-tier
sudo chown -R ec2-user:ec2-user /home/ec2-user/app-tier
sudo chmod -R 755 /home/ec2-user/app-tier

curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 16 && nvm use 16
npm install -g pm2
npm install && npm audit fix

pm2 start index.js
pm2 startup        # run the sudo command it prints
pm2 save
curl http://localhost:4000/health
```

**Web tier**

```bash
sudo -su ec2-user
cd /home/ec2-user
sudo aws s3 cp s3://<your-bucket>/application-code/web-tier web-tier --recursive
sudo chown -R ec2-user:ec2-user /home/ec2-user
sudo chmod -R 755 /home/ec2-user

curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 16 && nvm use 16
cd /home/ec2-user/web-tier
npm install
npm run build

sudo yum install nginx -y
cd /etc/nginx
sudo mv nginx.conf nginx-backup.conf
sudo aws s3 cp s3://<your-bucket>/application-code/nginx.conf .
sudo chmod -R 755 /home/ec2-user
sudo service nginx restart
sudo chkconfig nginx on
```

## Cleanup (to avoid AWS charges)

Delete in this order:

1. Auto Scaling Groups
2. Application Load Balancers (listeners first, then the load balancers)
3. Target Groups
4. Launch Templates
5. AMIs
6. Snapshots
7. EC2 instances
8. Aurora database (disable deletion protection first, skip final snapshot)
9. NAT Gateways
10. Elastic IPs
11. Route Tables (remove subnet associations first)
12. Internet Gateway
13. VPC
14. S3 bucket
15. IAM role

## Possible improvements

- Use AWS Secrets Manager instead of hardcoding DB credentials in `DbConfig.js`.
- Add a custom domain with Route 53 and HTTPS via ACM on the external ALB.
- Add target-tracking scaling policies (currently min = max = 2).
- Turn on ELB health checks in the Auto Scaling Groups.
- Provision everything with Terraform or CloudFormation.

## Credits

Project guide and source code by [Subodh Kumar](https://github.com/SubodhK143). Update this section if you adapt or extend the project.
