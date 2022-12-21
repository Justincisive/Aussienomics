---
title: Installing Ghost onto Oracle Cloud Always Free
ShowBreadCrumbs: false
ShowToc: false
author: Justin
date: "2022-12-09T18:29:00+08:00"
description: How to install ghost on Oracle's Always Free cloud server using Ubuntu. A step by step guide.
tags:
- tech
- off-topic
backtotop: true
toc: false
---

Have you ever wanted to run your own newsletter, with your own domain, but didn't want to fork out for a virtual private server (VPS) or a paid Substack plan? I certainly did; if you're like me, here's a simple how-to guide.

## Step 1: Buy a domain name
If you don't already own a domain name the cheapest place I've found is [Cloudflare](https://www.cloudflare.com/products/registrar/), which will also host your DNS for free.

## Step 2: Sign up for Oracle Cloud
Head over to [Oracle Cloud](https://www.oracle.com/cloud/free/) and register for an account. You'll need a credit card to complete the sign up process but you won't be charged, nor will you be charged in the future -- we're going to stay well within the limits of the *Always Free* plan.

If you use Brave browser, be sure to disable Brave Shields. If you don't you'll get errors later.

## Step 3: Create an instance
Once you have verified and signed into Oracle Cloud, visit the [Instances](https://cloud.oracle.com/compute/instances) page and click "Create instance".

Give it an appropriate name -- I used "Ghost", because I'm super original -- and then click the "Edit" button in the Image and shape section:

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

Make note of the public IP address and username. I'm not going to go into how to SSH to your server but if you're on Windows the easiest tool is [PuTTY](https://www.puttygen.com/download-putty). Just search around the internet for guides -- there are thousands -- about how to SSH into a Ubuntu server using PuTTY (you'll need those SSH keys we saved earlier).

When you create an instance with Oracle it's pretty bare bones. Ghost uses most of the 1GB of RAM you get on an Always Free instance, so the first thing we're going to do is create some swap space to stop it maxing out and hanging.

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

Now all we have to do is install a few Ghost prerequisites, such as a webserver, database, nodejs and opening some ports so people can actually visit the site:

`sudo apt-get install nginx`

`sudo apt-get install mysql-server`

`curl -sL https://deb.nodesource.com/setup_16.x | sudo -E bash`

`sudo apt-get install -y nodejs`

Once that's done, it's time to open some ports:

`sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 80 -j ACCEPT`

`sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 443 -j ACCEPT`

`sudo netfilter-persistent save`

Verify that your website works by visiting your public IP address, which should show a Welcome to nginx message: http://[your.public.ip.address]

## Step 6: Install Ghost
Before we install Ghost we need to make sure your domain is pointing to your instance. If you're using Cloudflare, create an A record, set the name as @ and the IPv4 address as your instance's public IP address. Make sure it's proxied so that Cloudflare can provide DDoS protection:

![Set your DNS](/images/ghost-oracle-7.png)

When your domain name propagates -- you'll see that nginx page from above when browsing to your domain -- we can proceed with the Ghost installation.

First, we need a MySQL database (be sure to change *YourPasswordHere*):

`sudo mysql`

`create user 'ghost'@'localhost' identified by 'YourPasswordHere';`

`grant all privileges on *.* to 'ghost'@'localhost';`

`create database ghost;`

`flush privileges;`

`exit`

Now we'll install the Ghost CLI:

`sudo npm install ghost-cli@latest -g`

The final step before we actually install Ghost is to create a folder for it and set the appropriate permissions:

`sudo mkdir -p /var/www/ghost`

`sudo chown ubuntu:ubuntu /var/www/ghost`

`sudo chmod 775 /var/www/ghost`

The Oracle servers are relatively slow, so we'll increase the timeout to allow us to actually install Ghost.

`sudo npm install --global yarn`

`yarn install --network-timeout 1000000`

Now to actually install Ghost:

`cd /var/www/ghost`

`ghost install`

Follow the prompts, using your own domain name, localhost, MySQL name (ghost), username/password from before (ghost/YourPasswordHere). You'll also want to say Yes when it asks you to let it configure nginx, set up SSL and systemd.

When it's all done and its starts up, congrats! You have a Ghost blog running on an Always Free Oracle instance.

## Step 7: Configure Ghost
I'm not going to go into a lot of detail here because everything can be found on the official [Ghost docs](https://ghost.org/docs/). But what you absolutely must do is set up SMTP email sending. 

You'll need to get yourself a Mailgun account (currently the only provider that can send Ghost newsletters) or some other transactional email sender and configure it to work with your domain. Once you've done that, open your config.production.json:

`cd /var/www/ghost`

`sudo nano config.production.json`

Then if you're using Mailgun, replace this:

```
{
  "mail": {
    "transport": "Direct"
  }
},
```

...with this:

```
  "mail": {
    "transport": "SMTP",
    "options": {
      "service": "Mailgun",
      "host": "smtp.mailgun.org",
      "port": 465,
      "secure": true,
      "auth": {
        "user": "postmaster@mail.yourdomain.com",
        "pass": "GeneratedStringHere"
      }
    }
  },
```

Then simply restart Ghost:

`ghost restart`

...aaaand you're done, congrats! If you want to optimise Ghost even further, read on 👇

## Step 7: Optimise Ghost

The first thing we'll do is reduce MySQL's memory usage:

`sudo nano /etc/mysql/conf.d/mysql.cnf`

Add the following:

```
[mysql]
# global settings
performance_schema = OFF

innodb_buffer_pool_size=50M
innodb_flush_method=O_DIRECT
innodb_log_buffer_size=1048576
innodb_log_file_size=4194304
innodb_max_undo_log_size=10485760
innodb_sort_buffer_size=64K
innodb_ft_cache_size=1600000
innodb_max_undo_log_size=10485760
max_connections=20
key_buffer_size=1M

# per-thread settings
thread_stack=140K
thread_cache_size = 2
read_buffer_size=8200
read_rnd_buffer_size=8200
max_heap_table_size=16K
tmp_table_size=128K
temptable_max_ram=2097152
bulk_insert_buffer_size=0
join_buffer_size=128
net_buffer_length=1K
```

Then restart MySQL:

`sudo service mysql restart`

Next, navigate to your ghost folder and create a file that will run every five hours to reduce the risk of running out of memory:

`cd /var/www/ghost`

`touch free-ram.sh && chmod +x free-ram.sh`

`nano free-ram.sh`

Then paste this into the file:

```
#!/bin/bash
FILE=/var/log/free-ram.log
sync
echo 3 > /proc/sys/vm/drop_caches

if test -f "$FILE"; then
    echo -e "$(date) \n$(egrep 'MemFree|MemAvailable' /proc/meminfo)"  >> /var/log/free-ram.log
else
    touch /var/log/free-ram.log
    echo -e "$(date) \n$(egrep 'MemFree|MemAvailable' /proc/meminfo)" >> /var/log/free-ram.log
fi
```

Now let's set it to run every five hours:

`sudo su`

`crontab -e`

Add this line to the bottom of the crontab file:

`0 */5 * * * /home/pi/projects/scripts/free-ram.sh`

Done! Type 'exit' to get out of superuser mode. Finally, we'll set up a file to rotate our logs to save on disk space:

`sudo nano /etc/logrotate.d/ghost`

Paste the following into the file:

```
/var/www/ghost/content/logs/*.log {
	weekly
	rotate 12
	compress
	copytruncate
}
```

Then restart Ghost just to make sure everything's in order:

`ghost restart`

That's all, folks! Ghost's logs will now rotate weekly and be stored for up to 12 weeks, RAM use has been optimised, and your swap file should ensure you don't run out of memory.