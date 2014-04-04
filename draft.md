---
layout: page
title: Setting Up a VPS for Ruby on Rails Hosting
---

Before developing the new version of [WinnipegElection.ca](http://winnipegelection.ca) I wanted to move the site from it's existing [Rackspace](http://www.rackspace.com) VPS to a fresh VPS with [Digital Ocean]( https://www.digitalocean.com/?refcode=9c57a647fd20). A VPS, by the way, is a [Virtual Private Server](https://en.wikipedia.org/wiki/Virtual_private_server), in our case a virtualized Linux server. On our existing VPS we were hosting close to 30 websites. Some of these sites were PHP, some Rails, and a few were even Perl. The idea was to create two VPSs, isolating all PHP sites on one and all Rails sites on the other (while decommissioning the Perl sites). Our decision to move from Rackspace to Digital Ocean was primary based on price and well as the simplicity of the Digital Ocean admin dashboard. The 512MB plan on [Digital Ocean]( https://www.digitalocean.com/?refcode=9c57a647fd20) is only $5 a month. Nice.

This post will detail the steps that were required to configure the new VPS for Ruby on Rails hosting. Our final hosting setup will be:

* Ubuntu 12.04 LTS
* Nginx with Phussion Passenger
* PostgreSQL
* Ruby 2.x
* Rails 4.x

### Step 1 - Creating the Droplet

Digital Ocean (DO) calls their VPS instances Droplets. Once you have a DO account you can create new droplets by clicking on the 'Create' button in your admin dashboard. I selected the following options:

* Size: 1GB\*
* Region: The default (New York 2)
* Linux Distribution: Ubuntu 12.04 LTS
* Settings: Enable VirtIO

\* *I started with a 1GB droplet to speed up the config process and later scaled back down to 512MB.*

### Step 2 - User Setup 

Once the droplet was created, I was emailed a root username and password. I was then able to login to the server as root using [Putty](http://www.chiark.greenend.org.uk/~sgtatham/putty/). My first order of business was to create a new user with sudoer previledges, and then lock root out of SSH access:

    adduser serveruser
    visudo

The second command opens up the sudoers file to which I added:

    serveruser    ALL=(ALL:ALL) ALL

I then edited the `/etc/ssh/sshd_config` file to add:

    PermitRootLogin no

And then I restarted ssh and logged back in with my new user:

    service ssh restart

From now on when I need to run a command as root I will proceed it with `sudo`.

### Step 3 - Creating a Swap File

Some of the configuration steps to follow require compilation and so it's nice to have a swap file on the server. By default DO droplets to not have swap files, but [adding one isn't difficult](https://www.digitalocean.com/community/articles/how-to-add-swap-on-ubuntu-12-04). I created a 1GB swapfile:

    sudo dd if=/dev/zero of=/swapfile bs=1024 count=1024k
    sudo mkswap /swapfile
    sudo swapon /swapfile
    sudo chown root:root /swapfile
    sudo chmod 0600 /swapfile

Then open `/etc/fstab` with sudo privs and add:

    /swapfile       none    swap    sw      0       0 

I then set the [swappiness](https://help.ubuntu.com/community/SwapFaq#What_is_swappiness_and_how_do_I_change_it.3F) to 25:

    echo 25 | sudo tee /proc/sys/vm/swappiness
    echo vm.swappiness = 20 | sudo tee -a /etc/sysctl.conf

### Step 4 - Installing Required Packages

Let's start by upgrading all the pre-installed packages:

    sudo apt-get update
    sudo apt-get upgrade

You may need to reboot after that:

    sudo reboot

Then we install all the packages we need 

    sudo apt-get install -y python-software-properties python
    sudo add-apt-repository ppa:chris-lea/node.js
    sudo apt-get update
    sudo apt-get install curl git g++ make nodejs libsqlite3-dev postgresql postgresql-contrib postgresql-server-dev-9.1 gconf2 libcurl4-openssl-dev

### Step 5 - PostgreSQL Setup

Switch to the postgres user:

    sudo su postgres

Add a database role/user with a password:

    createuser --pwprompt

Add a new database where `myapp` is the name of the Rails app you will be deploying:

    createdb myapp_development
    createdb myapp_test
    createdb myapp_production

### Step 6 - Installing Ruby and Rails

We are going to install Ruby by way of [rbenv](https://github.com/sstephenson/rbenv):

    curl https://raw.github.com/fesplugas/rbenv-installer/master/bin/rbenv-installer | bash

Add the supplied rbenv lines to .bashrc and source it.

    source ~/.bashrc

Get the required dependencies:

    rbenv bootstrap-ubuntu-12-04

Install Ruby (the latest version at the time of this post was 2.1.1):

    rbenv install 2.1.1
    rbenv global 2.1.1

Test your install with:

    ruby -v

And then we install Rails using gem:

    gem install rails

### Step 7 - Installing Passenger & NGINX:

First we install the [Phusion Passenger](https://www.phusionpassenger.com/) gem:

    gem install passenger 
  sudo passenger-install-nginx-module (Do a which passenger-install-nginx-module first and use that path.

Set up the startup scripts for NGINX:

  wget -O init-deb.sh http://library.linode.com/assets/660-init-deb.sh
  sudo mv init-deb.sh /etc/init.d/nginx
  sudo chmod +x /etc/init.d/nginx
  sudo /usr/sbin/update-rc.d -f nginx defaults

Start NGNIX (and then test using web browser):

  sudo service nginx start


To test, a simple /opt/nginx/conf/nginx.conf server block:

    server {
        listen       80;
        server_name  localhost;
        passenger_enabled on;
        rails_env development;
        root /var/www/blog/public;
    }

Create the /var/www folder and give it permissions.

  sudo groupadd www-pub
  sudo usermod -a -G www-pub vagrant 
  sudo chown -R stungeye:www-pub /var/www
  sudo chmod 2775 /var/www


Extra Security stuff:

  sudo apt-get install fail2ban
  https://www.digitalocean.com/community/articles/how-to-protect-ssh-with-fail2ban-on-ubuntu-12-04
  http://snippets.aktagon.com/snippets/554-how-to-secure-an-nginx-server-with-fail2ban

  Be cautious of the DDOS fail2ban, it could ban spiders like Googlebot!

Enable Firewall:

  sudo ufw default deny incoming
  sudo ufw default allow outgoing
  sudo ufw allow ssh
  sudo ufw allow www
  sudo ufw enable

Test out Rails:

in /var/www: rails new blog

Edit Gemfile, remove sqlite3 add pg.

Edit config/database.yml





