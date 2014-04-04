---
layout: page
title: Setting Up a VPS for Ruby on Rails Hosting
---



For VM SETUP:

  Command Prompt:
  
    vagrant init precise64 http://cloud-images.ubuntu.com/vagrant/precise/current/precise-server-cloudimg-amd64-vagrant-disk1.box

  Edit the Vagrantfile:

    config.ssh.forward_agent = true


  Port Forwarding in Vagrant:

    Add to the the Vagrantfile: config.vm.network :forwarded_port, guest: 80, host: 31337

  Login as root over SSH, add new user and add to sudoers.

    adduser stungeye
    adduser chefquix
    visudo
      Add to file:
        stungeye    ALL=(ALL:ALL) ALL
        chefquix    ALL=(ALL:ALL) ALL

If on VPS:

  Add a swapfile if required: https://www.digitalocean.com/community/articles/how-to-add-swap-on-ubuntu-12-04


Install required packages:

  sudo apt-get update
  sudo apt-get upgrade
  sudo apt-get install -y python-software-properties python
  sudo add-apt-repository ppa:chris-lea/node.js
  sudo apt-get update
  sudo apt-get install curl git g++ make nodejs libsqlite3-dev postgresql postgresql-contrib postgresql-server-dev-9.1 gconf2 libcurl4-openssl-dev

Setup Postgres

  sudo su postgres
  createuser --pwprompt            (And then create a new postgres role/user)
  createdb myapp_development       (Naming the db after your rails app name.)

(I created a mbelection:mbelection user with a manitobaelection_development, manitoba_election_production db)

Install rbenv:

  curl https://raw.github.com/fesplugas/rbenv-installer/master/bin/rbenv-installer | bash

Add the supplied rbenv lines to .bashrc and source it.

Get the required dependencies:

  rbenv bootstrap-ubuntu-12-04

Install Ruby 2.1.0:

  rbenv install 2.1.0
  rbenv global 2.1.0

(With Vagrant I had to disable https gem source to get the following to work: gem sources --remove https://rubygems.org/ && gem source --add http://rubygems.org)

Install Rails:

  gem install rails

Install Passenger & NGINX:

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





