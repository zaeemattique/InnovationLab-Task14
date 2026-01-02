# Task 14: Node.js Application Deployment on AWS EC2

## Overview
Deploy a Node.js application on AWS EC2 instances with automated CI/CD using GitHub Actions, Terraform IaC, and PM2 process management. Achieve high availability and rolling updates through Application Load Balancer and Auto Scaling Group.

## Architecture Components

![alt text](https://raw.githubusercontent.com/zaeemattique/InnovationLab-Task14/refs/heads/main/Task14%20Architecture%20Diagram.jpg)

- **VPC**: Custom networking with public/private subnets across 2 AZs
- **Compute**: EC2 instances managed by Auto Scaling Group
- **Load Balancing**: Application Load Balancer for traffic distribution
- **Storage**: S3 bucket for application artifacts
- **CI/CD**: GitHub Actions with OIDC authentication
- **Process Management**: PM2 for Node.js application lifecycle

## Deployment Workflow
```
Developer pushes code
    ↓
GitHub Actions builds app
    ↓
Artifact uploaded to S3
    ↓
ASG instance refresh triggered
    ↓
New EC2 instances launched
    ↓
User data pulls artifact from S3
    ↓
PM2 starts application
    ↓
ALB health check passes
    ↓
Old instances terminated
```

---

## Infrastructure Setup

### 1. Networking Infrastructure

**VPC Configuration**
- CIDR Block: `10.0.0.0/16`
- DNS Support & Hostnames: Enabled

**Subnets**
- Public Subnet A (us-west-2a): `10.0.1.0/24`
- Public Subnet B (us-west-2b): `10.0.3.0/24`
- Private Subnet A (us-west-2a): `10.0.2.0/24`
- Private Subnet B (us-west-2b): `10.0.4.0/24`

**NAT Gateways**
- NAT Gateway A in Public Subnet A
- NAT Gateway B in Public Subnet B

**Internet Gateway**
- Attached to VPC for public internet access

**Route Tables**
- Public RT: Route `0.0.0.0/0` → Internet Gateway (attached to Public Subnets)
- Private RT A: Route `0.0.0.0/0` → NAT Gateway A (attached to Private Subnet A)
- Private RT B: Route `0.0.0.0/0` → NAT Gateway B (attached to Private Subnet B)

**Security Groups**
- ALB SG: Allow inbound port 5000 from `0.0.0.0/0`
- EC2 SG: Allow all traffic from ALB SG, SSH on port 22

---

### 2. S3 Bucket for Artifacts

**Configuration**
- Name: `nodejs-artifacts-zaeem`
- Block all public access:
  - Block public ACLs: ✓
  - Block public policy: ✓
  - Ignore public ACLs: ✓
  - Restrict public buckets: ✓

---

### 3. IAM Configuration

**EC2 Instance Role**
- Service: `ec2.amazonaws.com`
- Permissions:
  - S3: `GetObject`, `ListBucket` on artifact bucket
  - CloudWatch Logs: Create log groups, streams, put log events
  - Auto Scaling: Describe ASG, launch configurations, instances
  - ELB: Describe target health, target groups

**GitHub Actions OIDC Role**
- Federated Identity: `token.actions.githubusercontent.com`
- Condition: Repository matching `repo:USERNAME/REPO:*`
- Permissions:
  - S3: `PutObject`, `GetObject`, `ListBucket` on artifact bucket
  - Auto Scaling: `StartInstanceRefresh`, `DescribeAutoScalingGroups`
  - IAM: `PassRole` for EC2 role

---

### 4. Application Load Balancer

**Target Group**
- Name: `Task14-ALB-Target-Group-Zaeem`
- Port: `5000`
- Protocol: `HTTP`
- Health Check:
  - Path: `/`
  - Interval: 30 seconds
  - Timeout: 5 seconds
  - Healthy threshold: 2
  - Unhealthy threshold: 2

**ALB Configuration**
- Name: `Task14-ALB-Zaeem`
- Scheme: Internet-facing
- Subnets: Public Subnet A & B
- Security Group: ALB SG

**Listener**
- Port: `5000`
- Protocol: `HTTP`
- Default Action: Forward to Target Group

---

### 5. Launch Template & Auto Scaling Group

**Launch Template**
- Name Prefix: `Task14-Launch-Template-Zaeem`
- AMI: Amazon Linux 2023
- Instance Type: `t3.micro`
- IAM Instance Profile: EC2 Instance Role
- Network: Public subnets with public IP
- Security Group: EC2 SG

**User Data Script**
```bash
#!/bin/bash
# Update system
dnf update -y

# Install Node.js, npm, unzip
dnf install -y nodejs npm unzip

# Install PM2 globally
npm install -g pm2

# Create app directory
mkdir -p /var/www/nodejs-app/current
chown -R ec2-user:ec2-user /var/www/nodejs-app

# Download artifact from S3
aws s3 cp s3://nodejs-artifacts-zaeem/app-latest.zip /tmp/app.zip

# Extract and set permissions
unzip /tmp/app.zip -d /var/www/nodejs-app/current
chown -R ec2-user:ec2-user /var/www/nodejs-app/current

# Install dependencies
cd /var/www/nodejs-app/current
sudo -u ec2-user npm install --production

# Start app with PM2
sudo -u ec2-user PORT=5000 pm2 start index.js --name nodejs-app

# Enable PM2 on reboot
sudo -u ec2-user pm2 save
sudo env PATH=$PATH:/usr/bin /usr/local/bin/pm2 startup systemd -u ec2-user --hp /home/ec2-user
```

**Auto Scaling Group**
- Name: `Task14-ASG-Zaeem`
- Desired Capacity: 2
- Min Size: 1
- Max Size: 3
- Subnets: Public Subnet A & B
- Target Group: Attach to ALB Target Group
- Instance Refresh: 50% minimum healthy percentage

---

## CI/CD Pipeline Configuration

### GitHub Actions Workflow

**File**: `.github/workflows/deploy.yml`

```yaml
name: Deploy to AWS

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-west-2
      
      - name: Build artifact
        run: |
          zip -r app-latest.zip . -x "*.git*" "node_modules/*"
      
      - name: Upload to S3
        run: |
          aws s3 cp app-latest.zip s3://nodejs-artifacts-zaeem/app-latest.zip
      
      - name: Trigger ASG instance refresh
        run: |
          aws autoscaling start-instance-refresh \
            --auto-scaling-group-name Task14-ASG-Zaeem \
            --preferences MinHealthyPercentage=50,InstanceWarmup=300
```

**GitHub Repository Secret**
- Name: `AWS_ROLE_ARN`
- Value: ARN of the GitHub Actions OIDC Role

---

## Application Requirements

**Required Files in Repository**
- `index.js` - Main application file
- `package.json` - Dependencies and metadata
- `package-lock.json` - Dependency lock file

**Application Configuration**
- Port: `5000`
- Environment: Production
- Process Manager: PM2

---

## Testing & Verification

### 1. GitHub Actions Pipeline
- Check workflow run status in GitHub Actions tab
- Verify artifact upload to S3
- Confirm ASG instance refresh triggered

### 2. Application Access
- Access via ALB DNS name: `http://<ALB-DNS-NAME>:5000`
- Verify load balancing across instances

### 3. EC2 Instance Verification
```bash
# SSH into instance
ssh ec2-user@<instance-ip>

# Check Node.js installation
node --version

# Check PM2 status
pm2 list
pm2 logs

# Test local application
curl localhost:5000

# Check user data logs
sudo cat /var/log/user-data.log
```

### 4. Health Checks
```bash
# Check target group health
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn>

# Check ASG instances
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names Task14-ASG-Zaeem
```

---

## Key Features

 **High Availability**: Multi-AZ deployment with ALB load balancing  
 **Rolling Updates**: ASG instance refresh with 50% minimum healthy  
 **Zero Downtime**: Health checks ensure new instances ready before old termination  
 **Automated CI/CD**: GitHub Actions with OIDC authentication  
 **Infrastructure as Code**: Terraform modules for reproducible deployments  
 **Process Management**: PM2 auto-restart and system startup configuration  
 **Security**: Private subnets, security groups, no hardcoded credentials

---

## Troubleshooting

**Common Issues**
1. **Node.js not installed**: Check user data script executed properly
2. **PM2 not starting**: Verify `index.js` exists and `ecosystem.config.js` (if used)
3. **Health checks failing**: Ensure app listens on port 5000
4. **OIDC authentication failed**: Verify GitHub repo matches trust policy
5. **S3 access denied**: Check EC2 IAM role permissions

**Logs to Check**
- `/var/log/cloud-init-output.log` - Cloud-init execution
- `/var/log/user-data.log` - User data script output
- `pm2 logs` - Application logs
- CloudWatch Logs - Centralized logging
