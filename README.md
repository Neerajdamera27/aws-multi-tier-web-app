# AWS Multi-Tier Employee Portal

A production-style, multi-tier web application deployed entirely on AWS — featuring a custom VPC, public/private subnet architecture, Bastion Host, Application Load Balancer, RDS (MySQL), DynamoDB, S3, SNS notifications, IAM roles, and Route 53 DNS routing.

---

## Architecture Overview

```
Internet
    │
    ▼
Route 53 (Custom Domain)
    │
    ▼
Application Load Balancer  ← public subnets (us-east-2a, us-east-2b)
    │
    ▼
Application Server (Flask + Apache)  ← private subnet
    │
    ├──► RDS MySQL (employee text data)      ← private subnet
    ├──► DynamoDB (employee image metadata)
    ├──► S3 Bucket (employee images)
    └──► SNS (upload notifications via email)

Bastion Host  ← public subnet (SSH gateway into private subnet)
```

---

## AWS Services Used

| Service | Purpose |
|---|---|
| **VPC** | Isolated network — `10.20.0.0/16` |
| **EC2** | Bastion Host (public) + Application Server (private) |
| **RDS (MySQL)** | Stores employee text records |
| **DynamoDB** | Stores image metadata (empid, S3 reference) |
| **S3** | Stores employee profile images |
| **ALB** | Exposes private app server to the internet |
| **NAT Gateway** | Allows private subnet to reach internet (outbound only) |
| **SNS** | Email alerts on every S3 upload |
| **IAM** | Role-based access for EC2 → S3, RDS, DynamoDB |
| **Route 53** | Maps custom domain to ALB via A record alias |

---

## Network Design

### VPC & Subnets — `10.20.0.0/16` (Ohio — us-east-2)

| Subnet | CIDR | Type | AZ |
|---|---|---|---|
| public-1 | `10.20.1.0/24` | Public | us-east-2a |
| private-1 | `10.20.2.0/24` | Private | us-east-2a |
| public-2 | `10.20.3.0/24` | Public | us-east-2b |
| private-2 | `10.20.4.0/24` | Private | us-east-2b |

> Two public subnets are required because the Application Load Balancer must span at least two Availability Zones.

### Routing

| Route Table | Destination | Target |
|---|---|---|
| `public-rt` | `0.0.0.0/0` | Internet Gateway |
| `private-rt` | `0.0.0.0/0` | NAT Gateway |

---

## Application Stack

- **Frontend:** HTML templates (Jinja2 via Flask)
- **Backend:** Python 3 + Flask (`EmpApp.py`)
- **Web Server:** Apache2 (port 80)
- **Database:** MySQL on RDS (`employee` table)
- **Object Storage:** S3 (employee images)
- **NoSQL:** DynamoDB (`employee_image_table`)


---

## Deployment Guide

### Prerequisites
- AWS account with Free Tier access
- EC2 Key Pair (.pem file)
- Region: **US East (Ohio) — us-east-2**

---

### Step 1 — VPC & Network Setup

1. Create VPC: `project-vpc` — CIDR `10.20.0.0/16`
2. Create 4 subnets (see Network Design table above)
3. Create and attach **Internet Gateway** (`project-IGW`) to VPC
4. Create **NAT Gateway** (`project-NAT`) in `public-1` subnet — allocate Elastic IP
5. Create `public-rt` → route `0.0.0.0/0` → IGW → associate `public-1`, `public-2`
6. Create `private-rt` → route `0.0.0.0/0` → NAT → associate `private-1`, `private-2`

---

### Step 2 — Security Groups & NACLs

- Create `project-sg` — inbound: All Traffic from Anywhere
- Verify NACLs: inbound rule 100 (Allow), outbound rule 100 (Allow), all subnets associated

---

### Step 3 — EC2 Instances (Ubuntu AMI)

| Instance | Subnet | Auto-assign Public IP |
|---|---|---|
| Bastion Host | public-1 | **Enable** |
| Application Server | private-1 | **Disable** |

---

### Step 4 — SSH into Application Server via Bastion

```bash
# From your local machine — copy key pair to Bastion Host
scp -i project.pem project.pem ubuntu@<bastion-public-ip>:/home/ubuntu/

# SSH into Bastion Host
ssh -i project.pem ubuntu@<bastion-public-ip>

# From Bastion — SSH into Application Server
chmod 400 project.pem
ssh -i project.pem ubuntu@<app-server-private-ip>
```

---

### Step 5 — Setup Application Server

```bash
sudo apt-get update -y

# Clone application code
sudo apt-get install git -y
git clone https://github.com/ZubairQuazi/aws-code-1.git

# Install Apache
sudo apt-get install apache2 -y
sudo mv aws-code-1/ /var/www/html/

# Install Python dependencies
sudo apt-get install python3-pip -y
sudo apt-get install python3-flask -y
sudo apt-get install python3-pymysql -y
sudo apt-get install python3-boto3 -y

# Install MySQL client
sudo apt-get install mysql-client -y
```

---

### Step 6 — Configure Application

```bash
cd /var/www/html/aws-code-1
sudo vim config.py
```

Update the following values:

```python
customhost     = "<RDS-endpoint>"
customuser     = "admin"
custompass     = "admin123"
customdb       = "employee"
custombucket   = "addemp--1"
customregion   = "us-east-2"
customdynamodb = "employee_image_table"
```

Also update `EmpApp.py` — find `region_name='us-west-2'` and change to `us-east-2`.

---

### Step 7 — RDS Setup (MySQL)

1. In RDS Console → **Subnet Groups** → create subnet group:
   - Name: `project`
   - VPC: `project-vpc`
   - AZs: us-east-2a, us-east-2b
   - Subnets: `10.20.2.0/24`, `10.20.4.0/24`

2. Create RDS instance:
   - Engine: MySQL | Tier: Free Tier
   - Credentials: `admin` / `admin123`
   - VPC: `project-vpc` | Subnet group: `project`
   - Public access: No | SG: `project-sg`
   - DB name: `employee` | Disable auto backup

3. Create the employee table:

```sql
mysql -h <rds-endpoint> -u admin -p

SHOW DATABASES;
USE employee;

CREATE TABLE employee (
  emp_id         VARCHAR(20),
  first_name     VARCHAR(20),
  last_name      VARCHAR(20),
  primary_skills VARCHAR(20),
  location       VARCHAR(20)
);

SHOW TABLES;
DESCRIBE employee;
EXIT;
```

---

### Step 8 — DynamoDB

- Table name: `employee_image_table`
- Primary key: `empid` (Number)

---

### Step 9 — S3 Bucket

- Bucket name: `addemp--1`
- Public access: **Enabled**

---

### Step 10 — Run the Application

```bash
cd /var/www/html/aws-code-1

# Stop Apache (both use port 80)
sudo systemctl stop apache2

# Run Flask app
sudo python3 EmpApp.py
```

---

### Step 11 — Application Load Balancer

1. Create **Target Group** → select Application Server
2. Create **ALB** (Application Load Balancer):
   - Scheme: Internet-facing
   - Subnets: `public-1` (us-east-2a), `public-2` (us-east-2b)
   - Target group: created above
3. Copy ALB DNS → paste in browser → app is accessible

---

### Step 12 — IAM Role (for S3/DynamoDB access)

1. IAM → Roles → Create Role → EC2
2. Permissions: `AdministratorAccess`, `AmazonEC2FullAccess`, `AmazonRDSFullAccess`
3. Role name: `project-role`
4. Attach to Application Server: EC2 → Actions → Security → Modify IAM Role

---

### Step 13 — Fix "Go Back" Button

```bash
cd /var/www/html/aws-code-1/templates

# Replace hardcoded IP with ALB endpoint
sudo nano AddEmpOutput.html   # update "action" attribute with ALB DNS
sudo nano GetEmpOutput.html   # remove hardcoded IP from "formaction"
```

---

### Step 14 — SNS Notifications

1. SNS → Create Topic → Standard → name: `event`
2. Create Subscription → Protocol: Email → enter your email → confirm from inbox
3. Edit Topic Access Policy — replace last two lines with:

```json
"ArnLike": {
  "aws:SourceArn": "arn:aws:s3:*:*:<your-bucket-name>"
}
```

4. S3 Bucket → Properties → Event Notifications → Create:
   - Event: All object create events
   - Destination: SNS topic `event`

---

### Step 15 — Route 53 (Custom Domain)

1. Register a free domain (e.g., via Freenom)
2. Route 53 → Create Hosted Zone → Public → enter domain name
3. Copy the 4 NS records → update in your domain registrar
4. Create Record:
   - Name: `www` | Type: A record | Alias: Yes → point to ALB
   - Routing policy: Simple routing

---

## Folder Structure

```
aws-multi-tier-web-app/
│
├── README.md
│
├── architecture/
│   └── architecture-notes.md
│
├── infra-setup/
│   ├── vpc-setup.md
│   ├── security-groups.md
│   └── iam-role.md
│
├── app-server-setup/
│   ├── bastion-host-commands.md
│   ├── app-server-commands.md
│   └── config-changes.md
│
├── database/
│   ├── rds-setup.md
│   ├── mysql-commands.sql
│   └── dynamodb-setup.md
│
├── services/
│   ├── s3-setup.md
│   ├── sns-setup.md
│   └── route53-setup.md
│
└── load-balancer/
    └── alb-setup.md
```

---

## Key Learnings

- Designed a **public/private subnet architecture** and understood why Bastion Hosts exist
- Configured a **NAT Gateway** for outbound-only internet from private subnets
- Deployed **RDS in a private subnet** with a custom DB subnet group
- Used **DynamoDB + S3 together** — structured metadata + raw file storage
- Implemented **IAM role-based access** instead of hardcoded credentials
- Connected **SNS to S3 events** for real-time email notifications
- Mapped a **custom domain to an ALB** using Route 53 alias A records

---

## Author

**Neeraj** — Junior Cloud Engineer | Hyderabad, India  
[GitHub](https://github.com/Neerajdamera27)
