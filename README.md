# Deploy Application on EC2 using AWS CodeDeploy & GitHub CI/CD Pipeline

> Reference: [YouTube Tutorial - Discover the Real AWS CodeDeploy | CICD Pipeline | Setup | Deploy application on EC2 using GitHub](https://www.youtube.com/watch?v=531i-n5FMRY&t=95s)

---

## Overview

This guide walks you through setting up a complete **CI/CD pipeline** using:

- **AWS CodeDeploy** — Automates application deployments to EC2 instances
- **GitHub** — Source code repository and trigger for deployments
- **Amazon EC2** — Target server where the application is deployed

---

## Architecture

```
GitHub (Push Code)
        |
        v
AWS CodePipeline (optional trigger)
        |
        v
AWS CodeDeploy
        |
        v
Amazon EC2 Instance
```

---

## Prerequisites

- An AWS account with admin or sufficient IAM permissions
- A GitHub account and a repository with your application code
- AWS CLI installed and configured locally
- Basic knowledge of Linux/EC2

---

## Step 1: Create IAM Roles

You need **two IAM roles**:

### 1a. EC2 Instance Role (allows EC2 to communicate with CodeDeploy and S3)

1. Go to **AWS Console → IAM → Roles → Create Role**
2. Select trusted entity: **AWS Service → EC2**
3. Attach the following policies:
   - `AmazonEC2RoleforAWSCodeDeploy`
   - `AmazonS3ReadOnlyAccess` *(if deploying from S3)*
4. Name the role: `EC2-CodeDeploy-Role`
5. Click **Create Role**

### 1b. CodeDeploy Service Role (allows CodeDeploy to call AWS services)

1. Go to **IAM → Roles → Create Role**
2. Select trusted entity: **AWS Service → CodeDeploy**
3. Attach the policy:
   - `AWSCodeDeployRole`
4. Name the role: `CodeDeploy-Service-Role`
5. Click **Create Role**

---

## Step 2: Launch an EC2 Instance

1. Go to **AWS Console → EC2 → Launch Instance**
2. Choose an AMI: **Amazon Linux 2** or **Ubuntu 22.04**
3. Select instance type: `t2.micro` (free tier eligible)
4. Under **IAM Instance Profile**, select `EC2Instance-CodeDeploy-Role`
5. Configure Security Group — allow:
   - **SSH** (port 22) from your IP
   - **HTTP** (port 80) from anywhere
6. Add a **Tag**:
   - Key: `Name`
   - Value: `MyAppServer` *(this tag is used by CodeDeploy to identify instances)*
7. Launch the instance and save the key pair (`.pem` file)

---

## Step 3: Install the CodeDeploy Agent on EC2

SSH into your EC2 instance and run the following commands:

### For Amazon Linux 2:
<!-- #!/bin/bash
sudo yum -y update
sudo yum -y install ruby
sudo yum -y install wget
cd /home/ec2-user
wget https://aws-codedeploy-ap-south-1.s3.ap-south-1.amazonaws.com/latest/install
sudo chmod +x ./install
sudo ./install auto
sudo yum install -y python-pip
sudo pip install awscli -->
```bash
sudo yum update -y
sudo yum install ruby wget -y

# Download the CodeDeploy agent installer
cd /home/ec2-user
wget https://aws-codedeploy-ap-south-1.s3.amazonaws.com/latest/install
# Replace 'ap-south-1' with your AWS region

chmod +x ./install
sudo ./install auto

# Start and enable the agent
sudo service codedeploy-agent start
sudo chkconfig codedeploy-agent on
```

### For Ubuntu:

```bash
sudo apt update -y
sudo apt install ruby wget -y

cd /home/ubuntu
wget https://aws-codedeploy-ap-south-1.s3.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto

sudo service codedeploy-agent start
sudo systemctl enable codedeploy-agent
```

### Verify the agent is running:

```bash
sudo service codedeploy-agent status
```

You should see: `The AWS CodeDeploy agent is running`

---

## Step 4: Prepare Your Application Code & appspec.yml

In your **GitHub repository**, your project structure should look like:

```
my-app/
├── appspec.yml
├── scripts/
│   ├── install_dependencies.sh
│   ├── start_server.sh
│   └── stop_server.sh
├── index.html          # or your app files
└── ...
```

### appspec.yml

This file is **required** by CodeDeploy. It tells the agent what to do during each deployment lifecycle event.

```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/html
    overwrite: true

hooks:
  BeforeInstall:
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: root

  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 300
      runas: root

  ApplicationStop:
    - location: scripts/stop_server.sh
      timeout: 300
      runas: root
```

### scripts/install_dependencies.sh

```bash
#!/bin/bash
yum install -y httpd   # For Amazon Linux
# OR
# apt install -y apache2   # For Ubuntu
```

### scripts/start_server.sh

```bash
#!/bin/bash
sudo service httpd start    # Amazon Linux
# OR
# sudo service apache2 start  # Ubuntu
```

### scripts/stop_server.sh

```bash
#!/bin/bash
isExistApp=$(pgrep httpd)
if [[ -n "$isExistApp" ]]; then
    sudo service httpd stop
fi
```

> Make scripts executable: `chmod +x scripts/*.sh`

---

## Step 5: Create a CodeDeploy Application

1. Go to **AWS Console → CodeDeploy → Applications → Create Application**
2. Enter:
   - **Application name**: `MyApp`
   - **Compute platform**: `EC2/On-premises`
3. Click **Create Application**

---

## Step 6: Create a Deployment Group

1. Inside your CodeDeploy application, click **Create Deployment Group**
2. Enter:
   - **Deployment group name**: `MyApp-DeploymentGroup`
   - **Service role**: Select `CodeDeploy-Service-Role` (created in Step 1)
3. **Deployment type**: `In-place`
4. **Environment configuration**: Select `Amazon EC2 instances`
   - Tag Key: `Name`, Tag Value: `MyAppServer`
5. **Deployment settings**: `CodeDeployDefault.OneAtATime`
6. **Load balancer**: Uncheck (disable) if not using one
7. Click **Create Deployment Group**

---

## Step 7: Connect GitHub to AWS CodeDeploy

1. In your CodeDeploy application, click **Create Deployment**
2. Select the deployment group: `MyApp-DeploymentGroup`
3. Under **Revision type**, select **GitHub**
4. Click **Connect to GitHub**
   - Authorize AWS CodeDeploy to access your GitHub account
   - Select the repository: `your-username/my-app`
5. Enter the **Commit ID** (latest commit hash from GitHub)
   - Go to GitHub → your repo → click the commit → copy the full SHA
6. Click **Create Deployment**

---

## Step 8: Monitor the Deployment

1. Go to **AWS Console → CodeDeploy → Deployments**
2. Click on your deployment ID (e.g., `d-XXXXXXXX`)
3. Monitor the lifecycle events:
   - `ApplicationStop`
   - `BeforeInstall`
   - `Install`
   - `AfterInstall`
   - `ApplicationStart`
   - `ValidateService`
4. Each event should show **Succeeded**

If any step fails, click on **View events** to see the error logs.

---

## Step 9: Verify the Deployment

1. Copy the **Public IPv4 address** of your EC2 instance
2. Open a browser and navigate to: `http://<EC2-Public-IP>`
3. You should see your deployed application

---

## Step 10: Automate with GitHub Actions (Optional CI/CD)

To trigger deployments automatically on every `git push`, create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to EC2 via AWS CodeDeploy

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1  # Change to your region

      - name: Create CodeDeploy Deployment
        run: |
          aws deploy create-deployment \
            --application-name MyApp \
            --deployment-group-name MyApp-DeploymentGroup \
            --github-location repository=${{ github.repository }},commitId=${{ github.sha }} \
            --ignore-application-stop-failures
```

### Add GitHub Secrets:

1. Go to your GitHub repo → **Settings → Secrets and variables → Actions**
2. Add:
   - `AWS_ACCESS_KEY_ID` — your IAM user's access key
   - `AWS_SECRET_ACCESS_KEY` — your IAM user's secret key

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| CodeDeploy agent not running | Run `sudo service codedeploy-agent restart` |
| Deployment stuck at `BeforeInstall` | Check script permissions (`chmod +x`) |
| `appspec.yml` not found | Ensure it is at the **root** of your repository |
| EC2 not found by deployment group | Verify EC2 tag matches the deployment group tag filter |
| GitHub connection fails | Re-authorize GitHub in CodeDeploy and check token permissions |
| Deployment fails with `No such file` | Verify `files` paths in `appspec.yml` match your repo structure |

---

## Key AWS Services Used

| Service | Purpose |
|---------|---------|
| **AWS CodeDeploy** | Automates application deployment |
| **Amazon EC2** | Hosts the deployed application |
| **IAM Roles** | Grants permissions to EC2 and CodeDeploy |
| **GitHub** | Source code repository and deployment trigger |
| **GitHub Actions** | CI/CD automation (optional) |

---

## Summary of Steps

1. Create **IAM roles** for EC2 and CodeDeploy
2. **Launch EC2** instance with the IAM role attached
3. **Install CodeDeploy agent** on the EC2 instance
4. Add **`appspec.yml`** and deployment scripts to your GitHub repo
5. Create a **CodeDeploy Application** in AWS Console
6. Create a **Deployment Group** targeting your EC2 instance
7. **Connect GitHub** to CodeDeploy and trigger a deployment
8. **Monitor** deployment lifecycle events in the console
9. **Verify** the app is live on your EC2 public IP
10. *(Optional)* Automate with **GitHub Actions** for full CI/CD
