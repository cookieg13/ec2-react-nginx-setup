# ec2-react-nginx-setup

This guide walks you through deploying a React Single Page Application (SPA) on an AWS EC2 instance using NGINX and securing it with SSL via Let's Encrypt. You'll also learn how to configure a custom domain (`dgspace.xyz`) and manage DNS using AWS Route 53.

---

## 🌐 Project Overview

- **Frontend**: React SPA (built with `npm run build`)
- **Server**: Ubuntu 22.04 EC2 Instance
- **Web Server**: NGINX
- **SSL**: Let's Encrypt via Certbot
- **Domain**: [`dgspace.xyz`](http://dgspace.xyz)

---

## 🚀 Prerequisites

- A built React app (`npm run build`)
- An AWS account (e.g., Lightsail or EC2)
- A domain (e.g., purchased from [GoDaddy](https://www.godaddy.com/))
- Access to your domain registrar's DNS settings
- `scp` and `ssh` installed locally
- Your EC2 PEM key (e.g., `dgtest.pem`)

---

## ☁️ EC2 Setup (Ubuntu-based)

### ✅ Step 1: Launch EC2 Instance

1. Go to [AWS EC2 Console](https://console.aws.amazon.com/ec2).
2. Choose a region close to your location.
3. Click "Launch Instance":
   - **Name**: `react-proxy-server`
   - **AMI**: Ubuntu Server 22.04 LTS
   - **Instance type**: `t2.micro` (Free Tier eligible)
   - **Key pair**: Create or upload (e.g., `dgtest.pem`)
   - **Network settings**:
     - Allow SSH (port 22) from your IP
     - Allow HTTP (port 80) from anywhere (0.0.0.0/0)
     - Click "Launch Instance"
     - Connect to Your Instance Wait for a minute until the instance is in "Running" state.
     - Click the instance and find the public IP (e.g. 12.34.56.78)

### ✅ Step 2: Connect to Your Instance



In your terminal:

```bash
mkdir -p ~/.ssh
mv ~/Downloads/dgtest.pem ~/.ssh/
cd ~/.ssh
xattr -c dgtest.pem
chmod -N dgtest.pem
chmod 400 dgtest.pem
````

SSH into the instance:

```bash
ssh -i ~/.ssh/dgtest.pem ubuntu@12.34.56.78
```

---

## 🌐 Install and Configure NGINX

```bash
sudo apt update
sudo apt install nginx -y
```

Check UFW status:

```bash
sudo ufw status
```

If active, allow HTTP traffic:

```bash
sudo ufw allow 80/tcp
```

Create NGINX config:

```bash
sudo nano /etc/nginx/sites-available/react-proxy.conf
```

Paste:

```nginx
server {
    listen 80;
    server_name dgspace.xyz www.dgspace.xyz;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Enable the config:

```bash
sudo ln -s /etc/nginx/sites-available/react-proxy.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

At this point, you can already visit http://12.34.56.78 (your EC2 IP) and see the NGINX welcome page or a test page.

Meanwhile, let's go set up your domain’s DNS.

---

## 🌍 Domain Setup (`dgspace.xyz`)

### Step 1: Buy Domain (e.g., GoDaddy)

Buy and manage your domain on [GoDaddy](https://www.godaddy.com/). After purchase:

* Go to **My Products** → Find your domain → Click **DNS / Manage DNS**

### Step 2: Delegate to AWS Name Servers

1. In AWS Route 53 → **Create a Hosted Zone**

   * Name: `dgspace.xyz`
   * Type: Public

2. Copy the 4 name servers listed under the `NS` record.

3. Back in GoDaddy:

   * Go to **DNS** → Change Nameservers → Enter custom nameservers.
   * Paste the 4 Route 53 nameservers and save.

> Name server propagation may take up to 1 hour.

### Step 3: Add A Records in Route 53

Go to your Route 53 hosted zone. Click “Create records.”

| Record Type | Name    | Value       | TTL |
| ----------- | ------- | ----------- | --- |
| A           | (blank) | 12.34.56.78 | 300 |
| A           | www     | 12.34.56.78 | 300 |

---
## 🔐 Enable HTTPS with Certbot

Install Certbot:

```bash
sudo apt install certbot python3-certbot-nginx -y
```

Run Certbot to configure SSL:

```bash
sudo certbot --nginx -d dgspace.xyz -d www.dgspace.xyz
```

Verify auto-renewal:

```bash
sudo certbot renew --dry-run
```

---

## 📦 Upload React Build to EC2

On your local machine, run:

```bash
scp -i ~/.ssh/dgtest.pem -r /path/to/react-app/build ubuntu@12.34.56.78:/home/ubuntu/dgspace
```

Then SSH into your server and confirm:

```bash
ssh -i ~/.ssh/dgtest.pem ubuntu@12.34.56.78
ls /home/ubuntu/dgspace
```

You should see the `build` folder inside.

---

## 🔧 Update NGINX for Production

Update the NGINX config:

```bash
sudo nano /etc/nginx/sites-available/react-proxy.conf
```

Replace with:

```nginx
server {
    listen 80;
    server_name dgspace.xyz www.dgspace.xyz;

    root /home/ubuntu/dgspace/build;
    index index.html;

    location / {
        return 301 https://$host$request_uri;
    }

    error_page 404 /404.html;
    location = /40x.html {
        root /usr/share/nginx/html;
    }
}

server {
    listen 443 ssl;
    server_name dgspace.xyz www.dgspace.xyz;

    root /home/ubuntu/dgspace/build;
    index index.html;

    ssl_certificate /etc/letsencrypt/live/dgspace.xyz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/dgspace.xyz/privkey.pem;

    location / {
        try_files $uri $uri/ /index.html;
    }

    error_page 404 /404.html;
    location = /40x.html {
        root /usr/share/nginx/html;
    }
}
```

Test and reload:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---


## ✅ Final Check

Visit:

* [http://dgspace.xyz](http://dgspace.xyz) → redirects to HTTPS
* [https://dgspace.xyz](https://dgspace.xyz) → shows your deployed React app

---

## 📝 Notes

* Always restart NGINX after modifying configs.
* SSL certificates expire in 90 days — Certbot will auto-renew if configured properly.
* Logs: `sudo tail -f /var/log/nginx/error.log`

---

## 📂 Folder Structure (on EC2)

```
/home/ubuntu/
├── dgspace/
│   └── build/
│       ├── index.html
│       ├── static/
│       └── ...
└── ...
```

---

## 📡 IP and Domain Reference

* **Public IP**: `12.34.56.78`
* **Domain**: `dgspace.xyz`


