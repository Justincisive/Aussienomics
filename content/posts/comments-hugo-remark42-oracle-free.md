---
title: Adding comments to Hugo with Remark42 and Oracle Cloud
ShowBreadCrumbs: false
ShowToc: false
author: Justin
date: "2023-02-09T13:14:00+08:00"
description: How to install Remark42 comments on Hugo using Oracle's Always Free cloud server and Ubuntu. A step by step guide.
tags:
- tech
- off-topic
backtotop: true
toc: false
---

If you read my previous articles on installing [Soapbox and Pleroma Rebased](/create-your-own-fediverse-instance-for-free/), or the [Ghost newsletter](/installing-ghost-onto-oracle-cloud-always-free/) software, on Oracle's Always Free cloud server using Ubuntu, then what follows should be somewhat familiar to you.

## Step 1: Buy a domain
I'm assuming your Hugo instance has its own domain name. If not, the cheapest place I've found is [Cloudflare](https://www.cloudflare.com/products/registrar/), which will also host your DNS for free. You'll need this because we're going to host your Remark42 comments on a subdomain.

## Step 2: Sign up for Oracle Cloud
Head over to [Oracle Cloud](https://www.oracle.com/cloud/free/) and register for an account. A credit card is required to complete the sign up process but you won't be charged, nor should you be charged in the future -- we're going to stay well within the limits of the *Always Free* plan.

If you use Brave browser, be sure to disable Brave Shields. If you don't you'll get errors later.

## Step 3: Create an instance
Once you have verified and signed into Oracle Cloud, visit the [Instances](https://cloud.oracle.com/compute/instances) page and click "Create instance".

Give it an appropriate name then click the "Edit" button in the Image and shape section:

![Creating your instance](/images/ghost-oracle-1.png)

Change the image from Oracle Linux to Canonical Ubuntu and click "Select image" down the bottom.

Once that's done, scroll down a bit further to "Add SSH keys" and either paste your existing public key or have Oracle generate one for you. Be sure to save a copy of your private and public keys, as you'll need them to SSH into your server later on.

The final step is to change the size of your instance. Scroll down to Boot volume and click "Specify a custom boot volume size".

You get up to two instances and 200GB for free, so if you want to create a second one at some point in the future set it to 100GB, otherwise go for the full 200GB. When you're done, click "Create":

![Picking a size](/images/ghost-oracle-2.png)

## Step 4: Connecting to your instance
Once Oracle is done provisioning your new instance, we need to open some ports so that it can talk to the outside world. On your Oracle Cloud dashboard, browse to Networking -> [Virtual Cloud Networks](https://cloud.oracle.com/networking/vcns), then click your VCN:

![Selecting your VCN](/images/ghost-oracle-5.png)

Continue clicking through the only options there, starting with subnet-\* and then Default Security List for vcn-\*, which will take you to your instance's Ingress Rules. Click "Add Ingress Rules" and add ports 80,443 to the "Destination Port Range", which when done should look something like this:

![Open ports](/images/ghost-oracle-6.png)

## Step 5: Configuring your instance
Head back to your [instance page](https://cloud.oracle.com/compute/instances) in Oracle, then click on its name to get all the details you'll need to SSH into it, most importantly its public IP address and username:

![Select your instance](/images/ghost-oracle-3.png)
![Copy your IP and username](/images/ghost-oracle-4.png)

Make note of the public IP address and username. The IP address is what you'll need to create your subdomain. Sign into Cloudflare, select DNS, then create an A record for something like comments.yourdomain.com and point it to the public IP you just copied.

Once that's done, you'll need to SSH into your new server. I'm not going to go into details about how do that but if you're on Windows the easiest tool is [PuTTY](https://www.puttygen.com/download-putty). Just search around the internet for guides -- there are thousands -- about how to SSH into a Ubuntu server using PuTTY (you'll need those SSH keys we saved earlier).

When you create an instance with Oracle it's pretty bare bones, so the first thing we're going to do is create some swap space to stop it maxing out and hanging.

To do that type the following commands, one by one, into your PuTTY session (big thanks to [Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-20-04) for the how-to):

`sudo fallocate -l 1G /swapfile`

`sudo chmod 600 /swapfile`

`sudo mkswap /swapfile`

`sudo swapon /swapfile`

`sudo cp /etc/fstab /etc/fstab.bak`

`echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab`

`sudo sysctl vm.swappiness=10`

`sudo sysctl vm.vfs_cache_pressure=50`

To make sure the swap is created every boot, type:

`sudo nano /etc/sysctl.conf`

Then add this to the bottom, save, and exit:

```
vm.swappiness=10
vm.vfs_cache_pressure=50
```

Now we'll update our instance with the following commands:

`sudo apt update`

`sudo apt upgrade`

Once that's done, it's time to open some ports:

`sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 80 -j ACCEPT`

`sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 443 -j ACCEPT`

`sudo netfilter-persistent save`

## Step 6: Installing Docker Engine
Now that the server is ready it's time to install Docker. These instructions are largely from [here](https://docs.docker.com/engine/install/ubuntu/).

**Remove any old versions**

`sudo apt-get remove docker docker-engine docker.io containerd runc`

Update our packages and install Docker:

`sudo apt-get update`

`sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release`

`sudo mkdir -p /etc/apt/keyrings`

`curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg`

`echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null`

`sudo apt update`

`sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`

All done! Verify it worked with this command:

`sudo docker run hello-world`

## Step 7: Configure Docker

Given that we're running this instance as ubuntu rather than root, we'll need to change the default unix socket user to prevent permission errors down the road:

`sudo nano /etc/systemd/system/sockets.target.wants/docker.socket`

Change the SocketUser from `root` to `ubuntu`, then exit (ctrl+o, ctrl+x, enter).

## Step 8: Install Remark42
For this next part we're going to be drawing heavily from the [official docs](https://remark42.com/docs/getting-started/installation/). Create the default config, making sure to change the variables to fit your instance.

`sudo nano docker-compose.yml`

```
version: '2'

services:
    remark:
        build: .
        image: umputun/remark42:latest
        container_name: "remark42"
        hostname: "remark42"
        restart: always

        logging:
            driver: json-file
            options:
                max-size: "10m"
                max-file: "5"

        # uncomment to expose directly (no proxy)
        ports:
          - "127.0.0.1:8080:8080"

    environment:
      - REMARK_URL=https://comments.yourdomain.com
      - SITE=comments
      - ADMIN_SHARED_EMAIL=admin@yourdomain.com
      - AUTH_EMAIL_ENABLE=true
      - AUTH_EMAIL_HOST=smtp.mailgun.org
      - AUTH_EMAIL_PORT=465
      - AUTH_EMAIL_FROM=noreply@comments.yourdomain.com #Change this to your site
      - AUTH_EMAIL_SUBJ=Confirm your email address
      - AUTH_EMAIL_CONTENT_TYPE=text/html
      - AUTH_EMAIL_TLS=true
      - AUTH_EMAIL_USER=postmaster@comments.yourdomain.com
      - AUTH_EMAIL_PASSWD=ReplaceThisWithARandomPassword # generated when a SMTP user is created
      - AUTH_EMAIL_TIMEOUT=10s
      - NOTIFY_EMAIL_FROM=noreply@comments.yourdomain.com
      - NOTIFY_EMAIL_ADMIN=true # notify admin on new comments
      - SMTP_HOST=smtp.mailgun.org
      - SMTP_PORT=465
      - SMTP_TLS=true
      - SMTP_USERNAME=postmaster@comments.yourdomain.com
      - SMTP_PASSWORD=ReplaceThisWithYourMailgunSMTPPassword
      - SECRET=ReplaceThisWithARandomSecret
      - DEBUG=true
      - AUTH_ANON=true
      - ADMIN_SHARED_ID=email_randomstringofadminuserhere # add this after you make your first comment as the administrator.
      # Enable it only for the initial comment
      # import or for manual backups.
      # Do not leave the server running with the
      # ADMIN_PASSWD set if you don't have an intention
      # to keep creating backups manually!
      # - ADMIN_PASSWD=<your secret password>
      #- SSL_TYPE=auto

        volumes:
            - ./var:/srv/var
```

Before we move on, check what Docker containers we're working with. It should just show the test one from earlier.

`docker container ls -a`

To install Remark42 we have to pull it and have it run in the background:

`docker-compose pull && docker-compose up -d `

Done! Now we'll set up Nginx so your comments are visible on the internet.

## Step 9: Install Nginx
First, create a config file and configure it. Once again, be sure to change `server_name` to your actual comments subdomain.

`sudo nano /etc/nginx/sites-available/comments.yoursite.com.conf`

```
server {

    listen 80;
    listen [::]:80;

    server_name comments.yoursite.com;

    location / {
         proxy_set_header        X-Forwarded-Proto $scheme;
         proxy_set_header        X-Real-IP $remote_addr;
         proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
         proxy_set_header        Host $http_host;
         proxy_pass              http://localhost:8080;
     }

}
```

Now enable it and restart Nginx:

`sudo ln -s /etc/nginx/sites-available/comments.yoursite.com.conf /etc/nginx/sites-enabled/comments.yoursite.com.conf`

`systemctl restart nginx`

Visit your test site to ensure it's live: http://comments.yoursite.com/web/

## Step 10: Encrypting traffic
Now we have to protect your comments (we don't want our guests' email addresses being sent in plain text!). Install Certbot:

`sudo certbot --nginx`

Follow the prompts. When you're done your site should be available via https: https://comments.yoursite.com/web/

## Step 11: Add comments to Hugo
This is the easiest part -- just paste the following into the post.hbs template where you want your comments to appear, ensuring that you change `host` and `site_id` to the values you set in docker-compose.yml earlier:

```
<script>
  var remark_config = {
    host: 'https://comments.yourdomain.com',
    site_id: 'comments',
  }
</script>
<script>!function(e,n){for(var o=0;o<e.length;o++){var r=n.createElement("script"),c=".js",d=n.head||n.body;"noModule"in r?(r.type="module",c=".mjs"):r.async=!0,r.defer=!0,r.src=remark_config.host+"/web/"+e[o]+c,d.appendChild(r)}}(remark_config.components||["embed"],document);</script>
  <div id="remark42"></div>
```

Done! Your Hugo posts should now have Remark42 comments.