# Deploy Next.js Application on Ubuntu with Nginx

This guide walks you through deploying a Next.js app on Ubuntu using Nginx as a reverse proxy, ideal for testing GitHub Actions CI/CD workflows.

## Prerequisites

- Ubuntu server (20.04 or later)
- Sudo privileges
- Domain name or server IP address
- GitHub repository with your Next.js app

## Step 1: Update System Packages

```bash
sudo apt update
sudo apt upgrade -y
```

## Step 2: Install Node.js and npm

Install Node.js using NodeSource repository:

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

Verify installation:

```bash
node --version
npm --version
```

## Step 3: Install PM2 Process Manager

PM2 keeps your Next.js app running in the background:

```bash
sudo npm install -g pm2
```

## Step 4: Create Application Directory

```bash
sudo mkdir -p /var/www/next-default
sudo chown -R $USER:$USER /var/www/next-default
cd /var/www/next-default
```

## Step 5: Clone Your Repository

```bash
git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git .
```

Or for testing, create a simple Next.js app:

```bash
npx create-next-app@latest . --typescript --tailwind --app --no-src-dir
```

## Step 6: Install Dependencies and Build

```bash
npm install
npm run build
```

## Step 7: Start Application with PM2

```bash
pm2 start npm --name "next-default" -- start
pm2 save
pm2 startup
```

Copy and run the command that PM2 outputs to enable startup on boot.

Verify the app is running:

```bash
pm2 status
pm2 logs next-default
```

## Step 8: Install and Configure Nginx

Install Nginx:

```bash
sudo apt install nginx -y
```

Create Nginx configuration:

```bash
sudo nano /etc/nginx/sites-available/next-default
```

Add the following configuration:

```nginx
server {
    listen 80;
    server_name your_domain.com;  # Replace with your domain or IP

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Step 9: Enable Nginx Configuration

```bash
sudo ln -s /etc/nginx/sites-available/next-default /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

## Step 10: Configure Firewall

```bash
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH
sudo ufw enable
```

## Step 11: Test Your Deployment

Visit your server IP or domain in a browser:
```
http://your_domain.com
```

## Step 12: Set Up GitHub Actions Deploy User

Create a dedicated deployment user:

```bash
sudo adduser deploy
sudo usermod -aG sudo deploy
```

Generate SSH key for GitHub Actions:

```bash
sudo su - deploy
ssh-keygen -t ed25519 -C "github-actions"
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
cat ~/.ssh/id_ed25519  # Copy this private key for GitHub Secrets
```

## Step 13: Create GitHub Actions Workflow

In your repository, create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Ubuntu

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: deploy
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            set -e

            cd /var/www/next-default

            git fetch origin
            git reset --hard origin/main
            git clean -fd

            sudo rm -rf .next node_modules
            sudo chown -R deploy:deploy /var/www/next-default
            npm ci --no-audit --no-fund
            npm run build

            pm2 restart next-default || pm2 start npm --name next-default -- start
            pm2 save

```

## Step 14: Configure GitHub Secrets

In your GitHub repository:
1. Go to Settings → Secrets and variables → Actions
2. Add secrets:
   - `SERVER_HOST`: Your server IP or domain
   - `SSH_PRIVATE_KEY`: The private key from Step 12

## Step 15: Test the Workflow

Push a change to your main branch:

```bash
git add .
git commit -m "Test deployment"
git push origin main
```

Watch the GitHub Actions tab in your repository to see the deployment in action.

## Useful Commands

**Check PM2 status:**
```bash
pm2 status
pm2 logs next-default
```

**Restart application:**
```bash
pm2 restart next-default
```

**Check Nginx status:**
```bash
sudo systemctl status nginx
sudo nginx -t
```

**View Nginx logs:**
```bash
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

## Optional: Add SSL with Let's Encrypt

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d your_domain.com
```

## Troubleshooting

**Port 3000 already in use:**
```bash
pm2 delete next-default
lsof -ti:3000 | xargs kill -9
pm2 start npm --name "next-default" -- start
```

**Permission denied errors:**
```bash
sudo chown -R $USER:$USER /var/www/next-default
```

**Nginx 502 Bad Gateway:**
- Check if Next.js is running: `pm2 status`
- Check PM2 logs: `pm2 logs next-default`
- Verify port 3000 is listening: `netstat -tlnp | grep 3000`

Your Next.js application is now deployed and ready for automated deployments via GitHub Actions!