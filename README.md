# AWS 3-Tier Architecture Project (High Availability & Fault Tolerance)

This project demonstrates how to build and deploy a **3-tier web application architecture** on **AWS** using best practices like **VPC isolation, private subnets, load balancers, IAM roles, and managed database (RDS)**.

---

## 📌 What is 3-Tier Architecture?

A **3-tier architecture** separates an application into three logical layers:

1. **Web Layer** – Handles user requests (Frontend)
2. **Application Layer** – Processes business logic (Backend API)
3. **Database Layer** – Stores application data

This design improves **security, scalability, availability, and fault tolerance**.

---

## 🛠 AWS Services Used

* **VPC** – Network isolation
* **EC2** – Web & App servers
* **IAM** – Secure access to AWS services
* **S3** – Store application code & configs
* **RDS (MySQL)** – Managed database
* **ALB (Application Load Balancer)** – External & Internal traffic routing

---

## 🗺 Architecture Overview

* Region: **Mumbai (ap-south-1)**
* 2 Availability Zones for High Availability
* Public subnets for Web Tier
* Private subnets for App Tier & DB Tier
* NAT Gateway for outbound internet access from private subnets

---

## 🚀 Step 1: Create VPC

**VPC Details**

* VPC Name: `Project-VPC`
* CIDR Block: `192.168.0.0/16`
* Availability Zones: 2
* Public Subnets: 2
* Private Subnets: 4
* NAT Gateway: Enabled (Zonal – 1 AZ)
* VPC Endpoints: None

### Public Subnets

* `Project-VPC-subnet-public1-ap-south-1a`
* `Project-VPC-subnet-public2-ap-south-1b`

### Private Subnets

**App Tier**

* `Project-VPC-subnet-App1-ap-south-1a`
* `Project-VPC-subnet-App2-ap-south-1b`

**Database Tier**

* `Project-VPC-subnet-DB1-ap-south-1a`
* `Project-VPC-subnet-DB2-ap-south-1b`

---

## 🔐 Step 2: Create Security Groups

| Security Group  | Purpose                | Inbound Rules                   |
| --------------- | ---------------------- | ------------------------------- |
| Web-ALB-SG      | External Load Balancer | HTTP (80) – Anywhere IPv4       |
| Web-SG          | Web EC2                | HTTP from Web-ALB-SG & VPC CIDR |
| App-SG          | App EC2                | TCP 4000 from VPC CIDR          |
| Internal-ALB-SG | Internal Load Balancer | HTTP from VPC CIDR              |
| RDS-SG          | MySQL DB               | MySQL (3306) from VPC CIDR      |

---

## 📦 Step 3: Create S3 Bucket

* Bucket Name: `kiot-project-bucket`
* Upload application code from GitHub

  * `application-code/app-tier/`
  * `application-code/web-tier/`
  * `application-code/nginx.conf`

---

## 👤 Step 4: Create IAM Role

* Role Name: `AmazonEC2RoleforSSM`
* Service: EC2
* Purpose: Allow EC2 to access S3 & Systems Manager

Attach this role to **Web Tier** and **App Tier EC2 instances**.

---

## 🗄 Step 5: Database Tier (RDS – MySQL)

### Create DB Subnet Group

* Name: `DB-SNGP`
* VPC: `Project-VPC`
* Subnets:

  * `Project-VPC-subnet-DB1-ap-south-1a`
  * `Project-VPC-subnet-DB2-ap-south-1b`

### Create RDS Instance

* Engine: MySQL
* Deployment: Single-AZ (Free Tier)
* Username: `admin`
* Password: `Admin#123`
* Security Group: `RDS-SG`

---

## ⚙ Step 6: App Tier Setup (Private EC2 – Amazon Linux 2023)

> **AMI Used:** Amazon Linux 2023 (Kernel 6.1)
>
> ⚠️ Amazon Linux 2023 uses **dnf** (not yum) and does **NOT** support `amazon-linux-extras`.

---

### Update OS

```bash
sudo dnf update -y
```

---

### Install MySQL Client (AL2023 Compatible)

```bash
sudo dnf install mysql -y
```

---

### Connect to RDS MySQL

```bash
mysql -h <DB-ENDPOINT> -u admin -p
```

---

### Create Database & Table

```sql
CREATE DATABASE webappdb;
USE webappdb;

CREATE TABLE IF NOT EXISTS transactions (
  id INT NOT NULL AUTO_INCREMENT,
  amount DECIMAL(10,2),
  description VARCHAR(100),
  PRIMARY KEY(id)
);

INSERT INTO transactions (amount, description)
VALUES ('400', 'groceries');

SELECT * FROM transactions;
```

Exit MySQL:

```sql
exit;
```

---

### Configure Application

Update DB details in:

```
application-code/app-tier/DbConfig.js
```

---

### Install Node.js (NVM) & PM2 – AL2023

```bash
curl -o- https://raw.githubusercontent.com/avizway1/aws_3tier_architecture/main/install.sh | bash
source ~/.bashrc

nvm install 16
nvm use 16
node -v
```

Install PM2:

```bash
npm install -g pm2
```

---

### Download App Code from S3

```bash
cd ~
sudo aws s3 cp s3://kiot-project-bucket/application-code/app-tier/ app-tier --recursive
cd app-tier
npm install
```

---

### Start Application Using PM2

```bash
pm2 start index.js
pm2 startup
pm2 save
```

---

### Verify Application Health

```bash
curl http://localhost:4000/health
```

---

## 🔁 Step 7: Internal Load Balancer (App Tier)

* Create **Internal ALB** in private subnets
* Attach **App EC2 instances** to target group
* Copy Internal ALB DNS

Update `nginx.conf`:

```nginx
location /api/ {
  proxy_pass http://<INTERNAL-ALB-DNS>:80/;
}
```

Upload updated file to S3.

---

## 🌐 Step 8: Web Tier Setup (Public EC2 – Amazon Linux 2023)

---

### Update OS

```bash
sudo dnf update -y
```

---

### Install Node.js (NVM)

```bash
curl -o- https://raw.githubusercontent.com/avizway1/aws_3tier_architecture/main/install.sh | bash
source ~/.bashrc

nvm install 16
nvm use 16
```

---

### Download Web Tier Code

```bash
aws s3 cp s3://kiot-project-bucket/application-code/web-tier/ web-tier --recursive
cd web-tier
npm install
npm run build
```

---

### Install Nginx (Correct for AL2023)

```bash
sudo dnf install nginx -y
```

Enable and start Nginx:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

---

### Configure Nginx

```bash
cd /etc/nginx
sudo rm -f nginx.conf
sudo aws s3 cp s3://kiot-project-bucket/application-code/nginx.conf .
```

Test and restart Nginx:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---

### Fix Permissions

```bash
sudo chmod -R 755 /home/ec2-user
```

---

## ✅ Final Verification

* Open **port 80** in Web Tier SG
* Access application using **Web EC2 public IP**
* Insert data via UI and verify DB entries

---

## 📘 Auto Scaling – Concepts

This section explains **Auto Scaling concepts in simple English**. Read this first to avoid confusion while following the steps.

---

### 🔹 What is Auto Scaling?

**Auto Scaling** automatically **adds or removes EC2 instances** based on load (traffic or CPU usage).

Example:

* Low traffic → Fewer EC2 instances
* High traffic → More EC2 instances created automatically

This helps with:

* High Availability
* Fault Tolerance
* Cost Optimization

---

### 🔹 Why Auto Scaling is Needed in 3-Tier Architecture

Without Auto Scaling:

* If one EC2 fails → application goes down
* Manual scaling is slow and risky

With Auto Scaling:

* If EC2 fails → new EC2 is launched automatically
* Traffic increases → more EC2s are added
* Traffic decreases → unused EC2s are terminated

---

### 🔹 Auto Scaling Group (ASG)

An **Auto Scaling Group** is a logical group of EC2 instances.

It defines:

* Minimum instances (Min)
* Desired instances (Desired)
* Maximum instances (Max)

Example:

```
Min = 2
Desired = 2
Max = 4
```

Meaning:

* Always at least 2 EC2s running
* Can scale up to 4 EC2s if load increases

---

### 🔹 Launch Template (LT)

A **Launch Template** is a blueprint for EC2 instances.

It contains:

* AMI (Amazon Linux 2023)
* Instance type
* Security Group
* IAM Role
* User-data script

Whenever ASG needs a new EC2, it uses this template.

---

### 🔹 Scaling Policies

Scaling policies decide **WHEN to scale**.

We use **Target Tracking Policy**:

* Metric: CPU Utilization
* Target: 50%

Meaning:

* CPU > 50% → scale out (add EC2)
* CPU < 50% → scale in (remove EC2)

---

### 🔹 Role of Load Balancer with Auto Scaling

Auto Scaling always works **with a Load Balancer**.

* Load Balancer distributes traffic
* Auto Scaling adds/removes EC2s
* Target Group keeps track of healthy instances

In this project:

* Web Tier → External ALB
* App Tier → Internal ALB

---

### 🔹 Auto Scaling in This Project

| Tier     | Subnets | Load Balancer | Auto Scaling |
| -------- | ------- | ------------- | ------------ |
| Web Tier | Public  | External ALB  | Yes          |
| App Tier | Private | Internal ALB  | Yes          |
| DB Tier  | Private | RDS (Managed) | No           |

> ❗ RDS is managed by AWS, so EC2 Auto Scaling is **not required** for DB tier.

---

## ⚙ Step 9: Configure Auto Scaling (High Availability & Scalability)

Auto Scaling ensures that the application automatically **adds or removes EC2 instances** based on traffic and keeps the application **highly available**.

We will configure Auto Scaling for:

* ✅ **Web Tier**
* ✅ **App Tier**

---

## 🔁 Auto Scaling – App Tier

### Step 9.1: Create Launch Template (App Tier)

1. Go to **EC2 → Launch Templates → Create launch template**
2. Template name: `App-Tier-LT`
3. AMI: **Amazon Linux 2023**
4. Instance type: `t2.micro` (Free tier)
5. Key pair: Select your key
6. Network settings:

   * Do NOT select subnet (Auto Scaling will handle it)
   * Security Group: `App-SG`
7. IAM Role: `AmazonEC2RoleforSSM`

### User Data (App Tier)

Paste the following **user-data script**:

```bash
#!/bin/bash
sudo dnf update -y
sudo dnf install mysql -y

# Install Node.js using NVM
curl -o- https://raw.githubusercontent.com/avizway1/aws_3tier_architecture/main/install.sh | bash
source /home/ec2-user/.bashrc

nvm install 16
nvm use 16
npm install -g pm2

# Download app code from S3
cd /home/ec2-user
aws s3 cp s3://kiot-project-bucket/application-code/app-tier/ app-tier --recursive
cd app-tier
npm install

# Start app
pm2 start index.js
pm2 startup
pm2 save
```

Click **Create launch template**.

---

### Step 9.2: Create Auto Scaling Group (App Tier)

1. Go to **EC2 → Auto Scaling Groups → Create Auto Scaling group**
2. Name: `App-Tier-ASG`
3. Launch template: `App-Tier-LT`
4. VPC: `Project-VPC`
5. Subnets:

   * `Project-VPC-subnet-App1-ap-south-1a`
   * `Project-VPC-subnet-App2-ap-south-1b`
6. Attach to **Internal Load Balancer target group**

### Capacity Settings

* Desired: 2
* Minimum: 2
* Maximum: 4

### Scaling Policy

* Policy type: Target tracking
* Metric: Average CPU Utilization
* Target value: 50%

Create Auto Scaling Group.

---

## 🔁 Auto Scaling – Web Tier

### Step 9.3: Create Launch Template (Web Tier)

1. Go to **EC2 → Launch Templates → Create launch template**
2. Template name: `Web-Tier-LT`
3. AMI: **Amazon Linux 2023**
4. Instance type: `t2.micro`
5. Security Group: `Web-SG`
6. IAM Role: `AmazonEC2RoleforSSM`

### User Data (Web Tier)

```bash
#!/bin/bash
sudo dnf update -y
sudo dnf install nginx -y

# Install Node.js
curl -o- https://raw.githubusercontent.com/avizway1/aws_3tier_architecture/main/install.sh | bash
source /home/ec2-user/.bashrc

nvm install 16
nvm use 16

# Download web code
cd /home/ec2-user
aws s3 cp s3://kiot-project-bucket/application-code/web-tier/ web-tier --recursive
cd web-tier
npm install
npm run build

# Configure Nginx
cd /etc/nginx
aws s3 cp s3://kiot-project-bucket/application-code/nginx.conf .
nginx -t
systemctl enable nginx
systemctl restart nginx
```

Create the launch template.

---

### Step 9.4: Create Auto Scaling Group (Web Tier)

1. Go to **EC2 → Auto Scaling Groups → Create Auto Scaling group**
2. Name: `Web-Tier-ASG`
3. Launch template: `Web-Tier-LT`
4. VPC: `Project-VPC`
5. Subnets:

   * `Project-VPC-subnet-public1-ap-south-1a`
   * `Project-VPC-subnet-public2-ap-south-1b`
6. Attach to **External Load Balancer target group**

### Capacity Settings

* Desired: 2
* Minimum: 2
* Maximum: 4

### Scaling Policy

* Policy type: Target tracking
* Metric: Average CPU Utilization
* Target value: 50%

Create Auto Scaling Group.

---

## ✅ Auto Scaling Verification

* Terminate one EC2 instance manually → ASG launches a new one
* Increase load → New instances auto-create
* Check **Target Group health** → All instances should be healthy

---

## ⭐ Key Features

* Multi-AZ High Availability
* Auto Scaling for Web & App tiers
* Internal & External Load Balancers
* Secure private networking
* Production-ready AWS architecture

No Route 53 used

---

## 📌 Conclusion

This project demonstrates a **production-style AWS 3-tier architecture** using EC2, ALB, RDS, IAM, and S3. It is suitable for **DevOps learning, interviews, and real-world practice**.

---
