# Deploy React App on AWS EC2 - Complete Beginner's Guide

## What is EC2?

EC2 (Elastic Compute Cloud) is a virtual server in AWS. Unlike S3 (storage), EC2 gives you a full computer where you can install software, run applications, and have complete control.

---

## Architecture Overview

```
GitHub Push → GitHub Actions → Build App → Deploy to EC2 → Nginx serves the app
```

---

## Step 1: Launch EC2 Instance

### 1.1 Go to EC2 Service

- AWS Console search bar → Type **"EC2"**
- Click on **"EC2"**

### 1.2 Launch Instance

- Click orange **"Launch instance"** button

### 1.3 Configure Instance

**Name and tags:**

- **Name**: `devops-react-app-server`

**Application and OS Images (Amazon Machine Image):**

- Select **"Ubuntu"** (Free tier eligible)
- **Ubuntu Server 24.04 LTS** (or latest version)
- Architecture: **64-bit (x86)**

**Instance type:**

- Select **"t2.micro"** (Free tier eligible - 1 vCPU, 1 GB memory)

**Key pair (login):** ⚠️ **IMPORTANT**

- Click **"Create new key pair"**
- **Key pair name**: `devops-react-app-key`
- **Key pair type**: RSA
- **Private key file format**: `.pem` (for Mac/Linux) or `.ppk` (for Windows with PuTTY)
- Click **"Create key pair"**
- **File will download automatically - SAVE IT SAFELY!**

**Network settings:**

- Click **"Edit"**
- **Auto-assign public IP**: Enable
- **Firewall (security groups)**: Create security group
- **Security group name**: `devops-react-app-sg`
- **Description**: Security group for React app server

**Inbound Security Group Rules - Add these:**

1. **SSH** (Already there)
   - Type: SSH
   - Port: 22
   - Source: My IP (or Anywhere for learning)
2. **HTTP** - Click "Add security group rule"
   - Type: HTTP
   - Port: 80
   - Source: Anywhere (0.0.0.0/0)
3. **HTTPS** - Click "Add security group rule"
   - Type: HTTPS
   - Port: 443
   - Source: Anywhere (0.0.0.0/0)

**Configure storage:**

- **8 GB gp3** (Free tier allows up to 30GB)
- Leave as default

### 1.4 Launch

- Review everything
- Click **"Launch instance"**
- Wait ~1 minute for instance to start
- Click **"View all instances"**

### 1.5 Get Instance Details

- Find your instance (should be "Running")
- Note down:
  - **Public IPv4 address** (e.g., 13.51.XXX.XXX)
  - **Instance ID**

---

## Step 2: Connect to EC2 Instance

### 2.1 Set Key Permissions (Mac/Linux)

Open terminal and run:

```bash
chmod 400 ~/Downloads/devops-react-app-key.pem
```

### 2.2 Connect via SSH

```bash
ssh -i ~/Downloads/devops-react-app-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

Replace `YOUR_EC2_PUBLIC_IP` with your actual IP.

Type `yes` when asked about authenticity.

You should now be inside your EC2 server! 🎉

---

## Step 3: Set Up the Server

Run these commands one by one in your EC2 terminal:

### 3.1 Update System

```bash
sudo apt update && sudo apt upgrade -y
```

### 3.2 Install Node.js

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

Verify:

```bash
node --version
npm --version
```

### 3.3 Install Nginx (Web Server)

```bash
sudo apt install -y nginx
```

Start Nginx:

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
```

Test: Open browser and go to `http://YOUR_EC2_PUBLIC_IP`
You should see "Welcome to nginx!"

### 3.4 Install Git

```bash
sudo apt install -y git
```

### 3.5 Create App Directory

```bash
sudo mkdir -p /var/www/react-app
sudo chown -R ubuntu:ubuntu /var/www/react-app
```

---

## Step 4: Configure Nginx

### 4.1 Create Nginx Configuration

```bash
sudo nano /etc/nginx/sites-available/react-app
```

Paste this configuration:

```nginx
server {
    listen 80;
    server_name YOUR_EC2_PUBLIC_IP;

    root /var/www/react-app;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

Save: `Ctrl+O` → Enter → `Ctrl+X`

### 4.2 Enable the Site

```bash
sudo ln -s /etc/nginx/sites-available/react-app /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

---

## Step 5: Manual First Deployment (Test)

Let's test manually first:

```bash
cd /var/www/react-app
git clone https://github.com/debnathdhanashri/Devops_Basic_App.git .
npm install
npm run build
cp -r dist/* /var/www/react-app/
```

Now visit: `http://YOUR_EC2_PUBLIC_IP`

You should see your React app! 🚀

---

## Step 6: Set Up Automated Deployment

### 6.1 Create Deployment Script on EC2

```bash
nano /home/ubuntu/deploy.sh
```

Paste this:

```bash
#!/bin/bash

# Navigate to app directory
cd /var/www/react-app

# Pull latest code
git pull origin main

# Install dependencies
npm install

# Build the app
npm run build

# Copy build files
cp -r dist/* /var/www/react-app/

# Restart nginx
sudo systemctl reload nginx

echo "Deployment completed successfully!"
```

Save and make executable:

```bash
chmod +x /home/ubuntu/deploy.sh
```

### 6.2 Allow ubuntu user to reload nginx without password

```bash
sudo visudo
```

Add this line at the end:

```
ubuntu ALL=(ALL) NOPASSWD: /bin/systemctl reload nginx
```

Save: `Ctrl+O` → Enter → `Ctrl+X`

---

## Step 7: Set Up SSH Keys for GitHub Actions

### 7.1 On EC2, generate SSH key

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/github_deploy_key -N ""
```

### 7.2 Add public key to authorized_keys

```bash
cat ~/.ssh/github_deploy_key.pub >> ~/.ssh/authorized_keys
```

### 7.3 Display private key (copy this)

```bash
cat ~/.ssh/github_deploy_key
```

Copy the ENTIRE output (including `-----BEGIN` and `-----END` lines)

---

## Step 8: Update GitHub Secrets

Go to GitHub repo → Settings → Secrets and variables → Actions

Add these NEW secrets:

| Name           | Value                            |
| -------------- | -------------------------------- |
| `EC2_HOST`     | Your EC2 public IP address       |
| `EC2_USERNAME` | `ubuntu`                         |
| `EC2_SSH_KEY`  | The private key you copied above |

---

## Step 9: Update GitHub Actions Workflow

The workflow file needs to be updated to deploy to EC2 instead of S3.

---

## Step 10: Test Automated Deployment

1. Make a change to your app
2. Commit and push to GitHub
3. GitHub Actions will automatically:
   - Build the app
   - SSH into EC2
   - Deploy the new version
4. Your app updates automatically!

---

## Accessing Your App

**Your app URL:**

```
http://YOUR_EC2_PUBLIC_IP
```

---

## Important Commands

**Connect to EC2:**

```bash
ssh -i ~/Downloads/devops-react-app-key.pem ubuntu@YOUR_EC2_IP
```

**Check Nginx status:**

```bash
sudo systemctl status nginx
```

**View Nginx logs:**

```bash
sudo tail -f /var/log/nginx/error.log
```

**Manual deployment:**

```bash
/home/ubuntu/deploy.sh
```

---

## Cost Information

- **EC2 t2.micro**: 750 hours/month free (first 12 months)
- After free tier: ~$8-10/month
- **Always stop** your instance when not using it to save money!

**Stop instance (not terminate):**

- EC2 Console → Select instance → Instance state → Stop

---

## Troubleshooting

**Can't connect via SSH:**

- Check security group allows port 22 from your IP
- Verify key file permissions: `chmod 400 key.pem`

**App not showing:**

- Check nginx: `sudo systemctl status nginx`
- Check if files exist: `ls -la /var/www/react-app`
- Check nginx config: `sudo nginx -t`

**Permission denied:**

- Check ownership: `sudo chown -R ubuntu:ubuntu /var/www/react-app`

---

## Next Steps

1. ✅ Get EC2 instance running
2. ✅ Manual deployment works
3. ✅ Set up automated CI/CD
4. 🔄 Add custom domain (optional)
5. 🔄 Add SSL certificate (HTTPS) (optional)

---

Good luck! 🚀
