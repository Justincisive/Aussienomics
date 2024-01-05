---
title: Create your own fediverse instance (for free!)
ShowBreadCrumbs: false
ShowToc: false
author: Justin Pyvis
authors: 
 - Justin Pyvis
date: 2022-12-25T09:14:00+08:00
description: How to install Soapbox and Pleroma Rebased on Oracle's Always Free cloud server using Ubuntu. A step by step guide.
tags:
  - tech
  - off-topic
backtotop: true
toc: false
---
Had enough of Twitter and Facebook and want to stake your claim on the fediverse, a social network of decentralised, autonomous servers running on open source software? Look no further! I'm going to show you how to get Soapbox (frontend) and Pleroma Rebased (backend) up and running on your own server, for free.

If you read my [previous article](/installing-ghost-onto-oracle-cloud-always-free/) on installing the Ghost newsletter software on Oracle's Always Free cloud server using Ubuntu, what follows should be somewhat familiar to you.

## Step 1: Buy a domain name
If you don't already own a domain name the cheapest place I've found is [Cloudflare](https://www.cloudflare.com/products/registrar/), which will also host your DNS for free.

## Step 2: Sign up for Oracle Cloud
Head over to [Oracle Cloud](https://www.oracle.com/cloud/free/) and register for an account. You'll need a credit card to complete the sign up process but you won't be charged, nor will you be charged in the future -- we're going to stay well within the limits of the *Always Free* plan.

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

Make note of the public IP address and username. I'm not going to go into how to SSH to your server but if you're on Windows the easiest tool is [PuTTY](https://www.puttygen.com/download-putty). Just search around the internet for guides -- there are thousands -- about how to SSH into a Ubuntu server using PuTTY (you'll need those SSH keys we saved earlier).

When you create an instance with Oracle it's pretty bare bones. Rebased uses most of the 1GB of RAM you get on an Always Free instance, so the first thing we're going to do is create some swap space to stop it maxing out and hanging.

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

Then install the prerequisites:

`sudo apt install git curl build-essential postgresql postgresql-contrib cmake libmagic-dev imagemagick ffmpeg libimage-exiftool-perl nginx certbot python3-certbot-nginx unzip libssl-dev automake autoconf libncurses5-dev fasttext`

Next, create a user to run the backend. We'll call it pleroma:

`sudo useradd -r -s /bin/false -m -d /var/lib/pleroma -U pleroma`

Before we move on to the installation we'll need to optimise postgresql (our database). We're running this on a very lightweight server so every little bit of memory matters!

To do that, find your postgresql folder. It will be something like `/etc/postgresql/14/main`. Then edit your config:

`sudo nano /etc/postgresql/14/main/postgresql.conf`

You're going to need to go through and find and set the following settings, uncommentating them (removing the '#') if necessary:

```
shared_buffers = 256MB
effective_cache_size = 768MB
maintenance_work_mem = 64MB
work_mem = 13107kB
```

Once you've done that, restart postgresql:

`sudo systemctl restart postgresql`

Now we can install Rebased, Elixir and Soapbox. Most of this next section is taken directly from the official docs [here](https://soapbox.pub/install/):

## Step 6: Installing Rebased

Download the Rebased source code with git:

`sudo git clone -b develop https://gitlab.com/soapbox-pub/rebased /opt/pleroma`

`sudo chown -R pleroma:pleroma /opt/pleroma`

Enter the source code directory, and become the pleroma user:

`cd /opt/pleroma`

`sudo -Hu pleroma bash`

(You should be the pleroma user in /opt/pleroma for the remainder of this section.)

Rebased uses the Elixir programming language (based on Erlang). It’s important we use a specific version of Erlang (24), so we’ll use the asdf version manager to install it.

Install asdf:

`git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.10.0`

`echo ". $HOME/.asdf/asdf.sh" >> ~/.bashrc`

`echo ". $HOME/.asdf/completions/asdf.bash" >> ~/.bashrc`

`exec bash`

`asdf plugin-add erlang`

`asdf plugin-add elixir`

Finally, install Erlang/Elixir:

`asdf install`

(This will take about 15 minutes. ☕)

When that's done, install basic Elixir tools for compilation:

`mix local.hex --force`

`mix local.rebar --force`

Fetch Elixir dependencies:

`mix deps.get`

Finally, compile Soapbox:

`MIX_ENV=prod mix compile`

(This will take about 10 minutes. ☕)

It's time to preconfigure our instance. The following command will set up some basics such as your domain name:

`MIX_ENV=prod mix pleroma.instance gen`

Rename the generated file so it gets loaded at runtime and exit as the pleroma user:

`mv config/generated_config.exs config/prod.secret.exs`

`exit`

Before moving on, we have one more optimisation to perform, as per [these instructions](https://docs-develop.pleroma.social/backend/configuration/postgresql/#disable-generic-query-plans):

`sudo nano /opt/pleroma/config/prod.secret.exs`

Add the following to your config:

```
config :pleroma, Pleroma.Repo,
  prepare: :named,
  parameters: [
    plan_cache_mode: "force_custom_plan"
  ]
```

You may have to append it to the end of the existing Pleroma.Repo block, like so (don't forget the commas on all lines except the last one):

```
config :pleroma, Pleroma.Repo,
  adapter: Ecto.Adapters.Postgres,
  username: "pleroma",
  password: "your-db-password-here",
  database: "pleroma",
  hostname: "localhost",
  prepare: :named,
  parameters: [
    plan_cache_mode: "force_custom_plan"
  ]
```

While there you may as well configure your mailer, if you have one, e.g. Mailgun (a full list of supported mailers can be found [here](https://github.com/swoosh/swoosh#adapters)):

```
config :pleroma, Pleroma.Emails.Mailer,
  adapter: Swoosh.Adapters.Mailgun,
  api_key: "your-api-key-here"
```

Now we'll execute our SQL file as the postgres user:

`sudo -Hu postgres psql -f config/setup_db.psql`

Then run the database migration as the pleroma user:

`sudo -Hu pleroma bash -i -c 'MIX_ENV=prod mix ecto.migrate'`

Copy the systemd service and start Soapbox:

`sudo cp /opt/pleroma/installation/pleroma.service /etc/systemd/system/pleroma.service`

`sudo systemctl enable --now pleroma.service`

If you've made it this far, congrats! You're very close to being done. Your Rebased server is running, and you just need to make it accessible to the outside world.

## Step 7: Getting online

The last step is to make your server accessible to the outside world. We'll achieve that by installing Nginx and enabling HTTPS support.

We'll use certbot to get an SSL certificate:

`sudo mkdir -p /var/lib/letsencrypt/`

`sudo certbot certonly --email <your@emailaddress> -d <your.domain> --nginx`

Replace `<your@emailaddress>` and `<your.domain>` with real values. Then copy the example Nginx configuration and activate it:

`sudo cp /opt/pleroma/installation/pleroma.nginx /etc/nginx/sites-available/pleroma.nginx`

`sudo ln -s /etc/nginx/sites-available/pleroma.nginx /etc/nginx/sites-enabled/pleroma.nginx`

You must edit this file:

`sudo nano /etc/nginx/sites-enabled/pleroma.nginx`

Change all occurrences of example.tld with your site's domain name. Use Ctrl+X, Y, Enter to save. Finally, enable and start Nginx:

`sudo systemctl enable --now nginx.service`

It's time to install Soapbox itself! First, get the latest build.

`sudo curl -L https://gitlab.com/soapbox-pub/soapbox/-/jobs/artifacts/develop/download?job=build-production -o soapbox.zip`

Then unzip it:

`sudo busybox unzip soapbox.zip -o -d /opt/pleroma/instance`

Refresh your website and you should see your brand new fediverse instance. All that's left to do is to create your admin account.

Create your first user with administrative rights with the following task:

`cd /opt/pleroma`

`sudo -Hu pleroma bash -i -c 'MIX_ENV=prod mix pleroma.user new <username> <your@emailaddress> --admin'`

Done! To upgrade your instance, do read the [official docs](https://soapbox.pub/install/). Enjoy the fediverse!