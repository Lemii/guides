# Securing a Lisk Node with Nginx and Certbot <!-- omit in toc -->

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Installation Process](#installation-process)
  - [Step 1: Installing Nginx](#step-1-installing-nginx)
  - [Step 2: Installing Certbot](#step-2-installing-certbot)
  - [Step 3: Allow HTTPS through firewall](#step-3-allow-https-through-firewall)
  - [Step 4: Obtaining an SSL certificate](#step-4-obtaining-an-ssl-certificate)
  - [Step 5: Redirect your domain to the node's API endpoint](#step-5-redirect-your-domain-to-the-nodes-api-endpoint)
    - [HTTP API configuration](#http-api-configuration)
    - [WebSocket API configuration](#websocket-api-configuration)
    - [Dual HTTP + WebSocket API configuration](#dual-http--websocket-api-configuration)
  - [Step 6 (optional): Certificate renewal](#step-6-optional-certificate-renewal)
  - [Done ✅](#done-)
- [Sources](#sources)
- [Support](#support)

---

## Introduction

This guide will help you to enable SSL on your node, making all HTTP and WebSocket communication encrypted. We will make use of Nginx and Certbot. Instead of Certbot, you could also use a service like Cloudflare.

**Nginx** is a web server which can also be used as a reverse proxy, load balancer, mail proxy and HTTP / WebSocket cache.

**Certbot** is an easy-to-use client that fetches a certificate from Let's Encrypt—an open certificate authority launched by the EFF, Mozilla, and others—and deploys it to a web server.

> PLEASE NOTE: This guide is made as easy as possible and does not go in-depth in regards to each individual topic. I highly recommend visiting the resources below to learn more about using Nginx and Certbot.

## Prerequisites

- Fully synced Link node
- Registered domain name
- DNS records of domain pointing to the public IP address of your node (both with and without **www.**). Set this up **before** continuing with the rest of the guide.

## Installation Process

### Step 1: Installing Nginx

Update local packages:

```sh
sudo apt update
```

Install nginx:

```sh
sudo apt install nginx
```

Ignore any `fail2ban` errors.

Allow traffic on port 80 to be able to set things up:

```sh
sudo ufw allow 'Nginx HTTP'
```

Verify firewall rules:

```sh
sudo ufw status
```

Check if Nginx is running correctly by navigating to your node's public IP in a browser (http://your_server_ip). You should see the splash screen: **Welcome to nginx!**

### Step 2: Installing Certbot

Add the repository:

```sh
sudo add-apt-repository ppa:certbot/certbot
```

If you're using 16.04, update first:

```sh
sudo apt-get update
```

Install Certbot Nginx package:

```sh
sudo apt install python-certbot-nginx
```

### Step 3: Allow HTTPS through firewall

Allow HTTPS traffic:

```sh
sudo ufw allow 'Nginx Full'
```

Remove redundant HTTP profile:

```sh
sudo ufw delete allow 'Nginx HTTP'
```

### Step 4: Obtaining an SSL certificate

Let certbot obtain the certificates for your domains (replace `dummydomain.com` with your domain):

```sh
sudo certbot --nginx -d dummydomain.com -d www.dummydomain.com
```

- Enter your email
- Accept the terms of service
- Decide on whether to share your email
- Choose any of the 2 options (we'll overwrite this later, anyway)

Visit your domain via https to verify that everything works (for example: https://www.dummydomain.com/)

### Step 5: Redirect your domain to the node's API endpoint

Open the nginx config file with nano:

```sh
sudo nano /etc/nginx/sites-available/default
```

Remove all contents from the config and add the following (replacing all occurrences of **dummydomain.com** with your own domain):

#### HTTP API configuration

One location block; all traffic will be proxied from dummydomain.com to localhost:8000.

```nginx
server {
  listen 443;
  server_name dummydomain.com www.dummydomain.com;
  ssl on;
  ssl_certificate /etc/letsencrypt/live/dummydomain.com/fullchain.pem; # managed by Certbot
  ssl_certificate_key /etc/letsencrypt/live/dummydomain.com/privkey.pem; # managed by Certbot
  ssl_verify_client off;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

  location / {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-NginX-Proxy true;
    proxy_pass http://localhost:8000/;
    proxy_ssl_session_reuse off;
    proxy_set_header Host $http_host;
    proxy_cache_bypass $http_upgrade;
    proxy_redirect off;
  }
}
```

#### WebSocket API configuration

One location block; all traffic will be proxied from dummydomain.com to localhost:8080.

```nginx
server {
  listen 443;
  server_name dummydomain.com www.dummydomain.com;
  ssl on;
  ssl_certificate /etc/letsencrypt/live/dummydomain.com/fullchain.pem; # managed by Certbot
  ssl_certificate_key /etc/letsencrypt/live/dummydomain.com/privkey.pem; # managed by Certbot
  ssl_verify_client off;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

  location /ws {
    proxy_pass http://localhost:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }
}
```

#### Dual HTTP + WebSocket API configuration

Two location blocks; all traffic from dummydomain.com/ws will be proxied to localhost:8080, all other traffic to dummydomain.com will be proxied to localhost:8000.

```nginx
server {
  listen 443;
  server_name dummydomain.com www.dummydomain.com;
  ssl on;
  ssl_certificate /etc/letsencrypt/live/dummydomain.com/fullchain.pem; # managed by Certbot
  ssl_certificate_key /etc/letsencrypt/live/dummydomain.com/privkey.pem; # managed by Certbot
  ssl_verify_client off;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

  location /ws {
    proxy_pass http://localhost:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }

  location / {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-NginX-Proxy true;
    proxy_pass http://localhost:8000/;
    proxy_ssl_session_reuse off;
    proxy_set_header Host $http_host;
    proxy_cache_bypass $http_upgrade;
    proxy_redirect off;
  }
}
```

> Be sure to use the ports that you have configured in your Lisk node

Save and exit with CTRL + X and confirm

Next, verify the syntax of the config file with:

```sh
sudo nginx -t
```

If the syntax is correct, reload Nginx to load the new configuration:

```sh
sudo systemctl reload nginx
```

You can now access your API via:

HTTPS: `https://dummydomain.com`  
WebSocket: `wss://dummydomain.com/ws`

### Step 6 (optional): Certificate renewal

You can let Certbot automatically renew the certificate by using a cron job.

First, open up the cron editor:

```sh
crontab -e
```

Next, add the following job to your crontab file:

```sh
23 23 * * * certbot renew --renew-hook 'service nginx reload' >> /var/log/letsencrypt/renew.log
```

Save and exit with CTRL + X and confirm

### Done ✅

Enjoy your secure API endpoint!

## Sources

Parts of the steps above are taken from the excellent in-depth guides linked below. Check them out if you'd like to know more about these topics:

[https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-18-04](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-18-04)
[https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04)

## Support

You can support my work by [voting for my delegate](lisk://delegates/vote?votes=lemii). Thanks in advance!
