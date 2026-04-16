# Deploy Index.html to EC2 using GitHub Actions + SSH

> Repo: https://github.com/Abhichowhan123/demo-code-deployment  
> Stack: Static HTML only — served by nginx on EC2 (free tier)

---

## How It Works

```
git push origin main  (your laptop)
        |
        v
GitHub Actions detects push
        |  SSH into EC2
        v
EC2: git pull origin main
        |
        v
nginx serves Index.html at http://<EC2-Public-IP>
```

---

## Step 1: Launch EC2 Instance (Free Tier)

1. Go to [AWS Console → EC2 → Launch Instance](https://console.aws.amazon.com/ec2)
2. Fill in:
   - **Name**: `MyAppServer`
   - **AMI**: `Ubuntu Server 22.04 LTS` (free tier eligible)
   - **Instance type**: `t2.micro` (free tier eligible)
   - **Key pair**: Click **Create new key pair** → name it `my-key` → download `my-key.pem`
3. Under **Network settings → Edit**:
   - Allow **SSH** (port 22) — Source: `0.0.0.0/0`
   - Allow **HTTP** (port 80) — Source: `0.0.0.0/0`
4. Click **Launch Instance**
5. Wait ~1 minute, then copy the **Public IPv4 address**

---

## Step 2: SSH into EC2 and Set Up the Server

Open your terminal on your laptop:

```bash
# Fix key file permissions (required on Mac/Linux)
chmod 400 ~/Downloads/my-key.pem

# SSH into EC2 (replace with your Public IPv4)
ssh -i ~/Downloads/my-key.pem ubuntu@<EC2-Public-IP>
```

Once inside EC2, run these commands:

```bash
# Update packages and install git + nginx
sudo apt update -y
sudo apt install -y git nginx

# Start nginx web server
sudo systemctl start nginx
sudo systemctl enable nginx

# Give ubuntu user ownership of the web folder
sudo chown -R ubuntu:ubuntu /var/www/html

# Remove default nginx page
rm /var/www/html/index.nginx-debian.html

# Clone YOUR repo directly into the web folder
git clone https://github.com/Abhichowhan123/demo-code-deployment.git /var/www/html
```

Now test it — open your browser and go to:
```
http://<EC2-Public-IP>
```
You should see your `Index.html` page live!

> **Note:** nginx looks for `index.html` (lowercase) by default.  
> Since your file is `Index.html` (capital I), either rename it in your repo to `index.html`  
> OR run this on EC2 to create a symlink:
> ```bash
> ln -s /var/www/html/Index.html /var/www/html/index.html
> ```

---

## Step 3: Generate SSH Deploy Key

On your **laptop** (not EC2), run:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/deploy_key -N ""
```

This creates two files:
- `~/deploy_key` — **private key** (goes to GitHub Secrets)
- `~/deploy_key.pub` — **public key** (goes to EC2)

### Add the public key to EC2:

```bash
# Print public key contents
cat ~/deploy_key.pub
```

Copy the output, then back on EC2 run:

```bash
echo "PASTE_PUBLIC_KEY_HERE" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### Test the deploy key works:

```bash
ssh -i ~/deploy_key ubuntu@<EC2-Public-IP> "echo connected"
```

You should see: `connected`

---

## Step 4: Add Secrets to GitHub

1. Go to your repo: https://github.com/Abhichowhan123/demo-code-deployment
2. Click **Settings → Secrets and variables → Actions → New repository secret**
3. Add these 3 secrets:

| Secret Name | Value |
|-------------|-------|
| `EC2_SSH_KEY` | Full contents of `~/deploy_key` (private key — include the `-----BEGIN...` and `-----END...` lines) |
| `EC2_HOST` | Your EC2 Public IPv4 address (e.g. `54.123.45.67`) |
| `EC2_USER` | `ubuntu` |

---

## Step 5: Create the GitHub Actions Workflow

On your laptop, inside your local clone of `demo-code-deployment`, create this file:

**File path:** `.github/workflows/deploy.yml`

```yaml
name: Deploy to EC2

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Deploy to EC2 via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd /var/www/html
            git pull origin main
```

Commit and push:

```bash
git add .github/workflows/deploy.yml
git commit -m "add github actions deploy workflow"
git push origin main
```

---

## Step 6: Test Auto-Deploy

Now every time you push any change to `main`, GitHub Actions will automatically SSH into EC2 and pull the latest code.

### To test:
1. Edit `Index.html` on your laptop — change something visible
2. Push to GitHub:
   ```bash
   git add Index.html
   git commit -m "update html"
   git push origin main
   ```
3. Go to your repo → **Actions tab** — watch the workflow run (~30 seconds)
4. Refresh `http://<EC2-Public-IP>` — your change is live

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Browser shows nginx default page | Run `rm /var/www/html/index.nginx-debian.html` on EC2 |
| Browser shows blank/404 | Check that `Index.html` exists in `/var/www/html` — run `ls /var/www/html` on EC2 |
| GitHub Actions fails with SSH error | Make sure `EC2_SSH_KEY` secret contains the **private key** (`~/deploy_key`, not `~/deploy_key.pub`) |
| `git pull` fails on EC2 | Make sure you cloned into `/var/www/html` in Step 2 |
| Port 80 not accessible | Check EC2 Security Group → Inbound rules → HTTP port 80 is allowed |

---

## Cost (Free Tier)

| Resource | Cost |
|----------|------|
| EC2 t2.micro | Free for 12 months (750 hrs/month) |
| GitHub Actions (public repo) | Free (unlimited minutes) |
| nginx | Free (open source) |
| **Total** | **$0** |

---

## Quick Reference

```bash
# SSH into your EC2
ssh -i ~/Downloads/my-key.pem ubuntu@<EC2-Public-IP>

# Check what files are being served
ls /var/www/html

# Manually pull latest code on EC2
cd /var/www/html && git pull origin main

# Check nginx status
sudo systemctl status nginx

# Restart nginx
sudo systemctl restart nginx
```
