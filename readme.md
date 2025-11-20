# deployment steps for nodejs application / backend complete with ssl certificate

## PART 1: CREATE AND ACCESS VM

## Step 1: Create VM on Azure Portal

1. Go to portal.azure.com
2. Click "Create a resource" → "Virtual Machine"

---

3. Fill in:

   - Resource Group: Create new → scraper-rg
   - VM Name: scraper-vm
   - Region: Choose nearest (e.g., East US, Central India)
   - Image: Ubuntu Server 22.04 LTS
   - Size: Standard_B2s (2 vCPUs, 4GB RAM)
   - Username: azureuser
   - SSH public key source: Generate new key pair
   - Key pair name: scraper-vm_key

4. Click "Review + Create" → "Create"
5. Download the .pem file when prompted and save it safely!

---

## Step 2: Configure Networking (Open Ports)

1. After VM is created, go to VM → "Networking" (left sidebar)
2. Click "Add inbound port rule" and add these 3 rules:

- Rule 1:

  1. Destination port: 22
  2. Name: SSH

- Rule 2:

  1. Destination port: 3000
  2. Name: App_Port

- Rule 3:

  1.  Destination port: 80
  2.  Name: HTTP

---

## Step 3: Get VM IP and Connect

1. Go to VM Overview page
2. Copy the Public IP address

On your Arch Linux machine:

```bash
# Move key to .ssh directory
mkdir -p ~/.ssh
mv ~/Downloads/scraper-vm_key.pem ~/.ssh/

# Set correct permissions
chmod 400 ~/.ssh/scraper-vm_key.pem

# Connect to VM (replace <VM_IP> with your actual IP)
ssh -i ~/.ssh/scraper-vm_key.pem azureuser@<VM_IP>
```

**_Type yes when asked about fingerprint._**

---

# PART 2: SETUP VM ENVIRONMENT

1. Step 4: Install Node.js 24
2. On VM:

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Node.js 24
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs

# Verify installation
node --version    # Should show v24.x.x
npm --version

# Install build tools and git
sudo apt install -y build-essential git
```

# PART 3: DEPLOY YOUR CODE

# Step 5: Create GitHub Personal Access Token (PAT)

1. Go to github.com → Your profile → Settings
2. Developer settings → Personal access tokens → Tokens (classic)
3. Generate new token (classic)
4. Settings:

```
Note: Azure VM Scraper
Expiration: 90 days (or No expiration)
Scopes: Check ✅ repo

Click Generate token
⚠️ COPY THE TOKEN (starts with ghp\_...) - You won't see it again!
```

# Step 6: Clone Your Private Repository

1. On VM:

```bash
# Clone using PAT (replace with your details)
git clone https://<GITHUB_USERNAME>:<YOUR_PAT>@github.com/<USERNAME>/<REPO_NAME>.git

# Example:
# git clone https://john:ghp_abc123xyz@github.com/john/scraper-app.git

# Go to project directory
cd <REPO_NAME>
```

# Step 7: Configure Git Credentials (Optional but Recommended)

```bash
# Store credentials for future pulls
git config --global credential.helper store
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
```

# Step 8: Setup and Test Your Application

```bash
# Install dependencies
npm install

# Build TypeScript
npm run build

# Create .env file (if you have environment variables)
nano .env
```

**_note : In nano editor, add your variables_**

Test run your app:
`node dist/index.js`

`Test in browser: Open http://<VM_IP>:3000
Stop the app: Press Ctrl+C`

# PART 4: PRODUCTION SETUP

# Step 9: Install and Configure PM2

```bash
# Install PM2 globally
sudo npm install -g pm2

# Start your app with PM2
pm2 start dist/index.js --name scraper-app

# Make PM2 start on system reboot
pm2 startup systemd

# ⚠️ IMPORTANT: Copy and run the command that PM2 shows
# It looks like: sudo env PATH=$PATH:/usr/bin ...

# Save PM2 process list
pm2 save

# Check app status
pm2 status

# View logs
pm2 logs scraper-app
```

# Step 10: Install and Configure Nginx

```bash
# Install Nginx
sudo apt install -y nginx

# Create Nginx configuration file
sudo nano /etc/nginx/sites-available/scraper
```

**_Paste this configuration:_**

```bash
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;

        # Timeouts (important for scrapers)
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

# step 11

### Enable Nginx configuration:

```bash

# Create symbolic link
sudo ln -s /etc/nginx/sites-available/scraper /etc/nginx/sites-enabled/

# Remove default configuration
sudo rm /etc/nginx/sites-enabled/default

# Test Nginx configuration
sudo nginx -t

# If test passes, restart Nginx
sudo systemctl restart nginx

# Enable Nginx to start on boot
sudo systemctl enable nginx
```

---

---

# Add SSL Certificate to Your Azure DNS Name

# Step 1: Open Port 443 (HTTPS) on Azure

1. Go to Azure Portal → Your VM → Networking
2. Add inbound port rule:

- Destination port: 443
- Protocol: TCP
- Priority: 1003
- Name: HTTPS

# step 2 ssh into vm

```bash
ssh -i ~/.ssh/scraper-vm_key.pem azureuser@<VM_IP>
```

# Step 3: Install Certbot

```bash
# Update and install Certbot
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

# Step 4: Update Nginx Configuration

**Edit Nginx config**

```bash
sudo nano /etc/nginx/sites-available/scraper
```

```bash
server {
    listen 80;
    server_name scrapper-app.eastasia.cloudapp.azure.com;  # ← Change this

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;

        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
```

**Save: Ctrl+X, Y, Enter**

```bash
sudo nginx -t
sudo systemctl reload nginx
```

# Step 5: Get SSL Certificate

```bash
sudo certbot --nginx -d scrapper-app.eastasia.cloudapp.azure.com
```

# Step 7: Verify Auto-Renewal

```bash
sudo systemctl status certbot.timer

# Test renewal (dry run - doesn't actually renew)
sudo certbot renew --dry-run
```
