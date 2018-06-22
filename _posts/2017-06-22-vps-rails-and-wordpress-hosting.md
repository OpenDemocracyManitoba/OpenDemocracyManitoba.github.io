---
layout: page
title: VPS Wordpress & Rails Hosting
---

*ODM Technology Post*

**In this post we will configure Rails and Workpress on a [Digital Ocean](https://m.do.co/c/9c57a647fd20) VPS.**

This is an update to [a similar post from two years ago](/2015/01/10/vps-ruby-on-rails-hosting-2/). 

Our final hosting setup will be:

- OS: Linux - [Ubuntu](http://ubuntu.com/) LTS Version
- Web Server: [Nginx](http://nginx.org/)
- Rails Hosting: [Phussion Passenger](https://www.phusionpassenger.com/)
- Database Server: [MariaDb](https://mariadb.com/)
- Language and Framework: [Ruby](http://www.ruby-lang.org/) 1.87 (legacy) & 2.x, plus [Rails](http://rubyonrails.org/) 5.x
- Language and Framework: [PHP][http://php.net] 7.x and [Wordpress](http://wordpress.org) 4.9.x

**NOTE: ** Used MariaDB instead of Postgres because we need to install Wordpress.

*This tutorial assumes a familiarity with the Linux command line, server administration, and Rails configuration.*

### Background Information

A VPS is a [Virtual Private Server](https://en.wikipedia.org/wiki/Virtual_private_server). In our case, a virtualized Ubuntu Linux server from [Digital Ocean](https://m.do.co/c/9c57a647fd20) to host [ODM](http://opendemocracymanitoba.ca) Ruby projects.

The 1GB plan on [Digital Ocean](https://m.do.co/c/9c57a647fd20) is only $5 a month. Nice.

### Automation and Config Management

This post does not rely on a configuration management tool.

I figured I'd be done setting up the server long before I'd figured out the intricacies of [Puppet](http://puppetlabs.com/), [Chef](http://www.getchef.com/), or [Ansible](http://www.ansible.com). I tend to do a big deploy like this once every two years, often with significant changes required to the workflow, so I'm glad I haven't automated the process yet.

That said, if you can see yourself setting up multiple servers following this tutorial, you may wish to look into one of the automated provisioning solution listed in the previous paragraph.

### Step 1 - Create the Droplet

Digital Ocean (DO) calls their VPS instances Droplets. Once you have a DO account you can create new droplets by clicking on the 'Create' button in your admin dashboard. I selected the following options:

- Size: 2GB
- Region: Toronto 1
- Linux Distribution: Ubuntu 18.04 LTS

### Step 2 - User Setup

Once the droplet was created, I was emailed a root username and password. I was then able to SSH to the server as root using [Putty](http://www.chiark.greenend.org.uk/~sgtatham/putty/). My first order of business was to create a new user with sudoer privileges, and then lock root out of SSH access:

``` 
adduser serveruser
visudo
```

The second command opens up the sudoers file to which I added:

``` 
serveruser    ALL=(ALL:ALL) ALL
```

I then edited the `/etc/ssh/sshd_config` file to add:

``` 
PermitRootLogin no
```

And then I restarted ssh and logged back in with my `serveruser` user:

``` 
sudo service ssh restart
```

From now on when I need to run a command as root I will proceed the command with `sudo`. 

Not shown in this tutorial: [How to use SSH keys with Digital Ocean Droplets](https://www.digitalocean.com/community/articles/how-to-use-ssh-keys-with-digitalocean-droplets).

### Step 3 - Create a Swap File (Optional)

Some of the following steps require compilation, so it's nice to have a swap file on the server. By default DO droplets to not have swap files, but [adding one isn't difficult](https://www.digitalocean.com/community/tutorials/how-to-add-swap-on-ubuntu-14-04). This steps was much more necessary when I was using droplets with only 512MB RAM so feel free to skip this step.

I created a 1GB swapfile, half my physical RAM, since I'm going to dial the swappiness way down:

``` 
sudo fallocate -l 1G /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo chown root:root /swapfile
sudo chmod 0600 /swapfile
```

Then open `/etc/fstab` with sudo privs and add:

``` 
/swapfile       none    swap    sw      0       0 
```

Using swap too aggressively on a SSD drive can lead to hardware degradation. So let's dial down the "[swappiness](https://help.ubuntu.com/community/SwapFaq#What_is_swappiness_and_how_do_I_change_it.3F)" from 60 to 5:

``` 
echo 5 | sudo tee /proc/sys/vm/swappiness
echo vm.swappiness = 5 | sudo tee -a /etc/sysctl.conf
```

### Step 4 - Install Required Ubuntu Packages

Let's start by upgrading all the pre-installed packages:

``` 
sudo apt-get update
sudo apt-get upgrade
```

Reboot after that:

``` 
sudo reboot
```

Then install the ubuntu packages we need 

``` 
sudo apt-get install git g++ make nodejs libsqlite3-dev postgresql postgresql-contrib gconf2 libcurl4-openssl-dev imagemagick zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev sqlite3 libxml2-dev libxslt1-dev libffi-dev php-fpm php-curl php-imagick php-mbstring php-zip  php-mysql  subversion autoconf bison mariadb-server mariadb-client
```

### Step 5 - MariaDB Setup

[MariaDB](https://mariadb.com/) is a fork of the MySQL project. I normally use Postgres when working with Rails, but because I needed to support Wordpress too, I used Maria.

Switch to the root user:

``` 
sudo su -
```

Secure the database (press ENTER for the current password):

```
sudo mysql_secure_installation
```

Login with the root password you just set:

```
sudo mariadb -uroot -p
```

Add a new database from the SQL prompt:

```
create database newdatabasename;
```

Create a new user with full permissions to this database. Also from the SQL prompt:

```
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, ALTER ON newdatabasename.* TO 'newusername'@'localhost' IDENTIFIED BY 'newuserpassword';
```

(Be sure to replacce the `newdatabasename`, `newusername` and `newuserpassword` with values of your own.)

Exit the SQL prompt and switch back to the `serveruser`:

``` 
exit
exit
```

### Step 6 - Install Ruby and Rails

We are going to install Ruby by way of [rbenv](https://github.com/sstephenson/rbenv).

``` 
git clone https://github.com/rbenv/rbenv.git ~/.rbenv

echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
source ~/.bash_profile
cd ~/.rbenv && src/configure && make -C src
mkdir -p "$(rbenv root)"/plugins
git clone https://github.com/rbenv/ruby-build.git "$(rbenv root)"/plugins/ruby-build
~/.rbenv/bin/rbenv init
echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
source ~/.bash_profile
```

Install Ruby (the latest version at the time of this post was 2.5.1):

``` 
rbenv install 2.5.1
rbenv global 2.5.1
```

Test your install with version switch:

``` 
ruby -v
```

And then install bundler using gem:

``` 
echo "gem: --no-document" > ~/.gemrc
gem install bundler
```

I also need to install Ruby 1.8.7 for legacy reasons. The apt-get is a fix for openssl support.

```
sudo apt-get install libssl1.0-dev
CONFIGURE_OPTS="--with-readline-dir=/usr/include/readline --with-openssl-dir=/usr/include/openssl" rbenv install 1.8.7-p375
```

### Step 7 - Install Passenger & Nginx:

Install [Phusion Passenger](https://www.phusionpassenger.com/) and the [Nginx](http://nginx.org/) web server:

``` 
# Install our PGP key and add HTTPS support for APT
sudo apt-get install -u dirmngr gnupg
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 561F9B9CAC40B2F7
sudo apt-get install -y apt-transport-https ca-certificates

# Add our APT repository
sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger bionic main > /etc/apt/sources.list.d/passenger.list'
sudo apt-get update

# Install Passenger + Nginx
sudo apt-get install -y nginx-extras libnginx-mod-http-passenger
```

Re-start Nginx:

``` 
sudo service nginx restart
```

### Step 8 - File Permissions and Other Config

Create the /var/www folder and give it permissions. If /var/www already exists you can skip the `mkdir` step.

``` 
mkdir /var/www
sudo groupadd www-data
sudo usermod -a -G www-data serveruser 
sudo chown -R serveruser:www-data /var/www
sudo chmod 2775 /var/www
```

Set up a ssh key for the server:

``` 
ssh-keygen -t rsa -b 4096 -C "stungeye@gmail.com"
```

Configure git:

``` 
git config --global user.email "youremail@example.com"
git config --global user.name "Your Name"
```

Not shown in this tutorial: [Linking the SSH key created above to your Github account](https://help.github.com/articles/generating-ssh-keys#platform-linux).

### Step 9 - Configure Nginx to Host a Rails App

To test a Rails app, here's simple ngnix server block that you can put in your `/etc/nginx/sites-available/default` config file. You will need to edit this file using `sudo` and you should replace the existing server block that is already present in the file.

``` 
server {
    listen       80;
    server_name  localhost;
    passenger_enabled on;
    rails_env development;
    root /var/www/winnipegelection/public;
}
```

Note that the folder you specific for the `root` must point to the `public` folder of the app you cloned into the `/var/www/` folder. If you've mapped your droplet to a domain, replace `localhost` with the name of your domain.

You'll also want to configure your Rails app to use MySQL using the credentials you created above. And don't forget to run your migrations!

If you're hosting multiple sites you might want to use the [ensite tool](https://github.com/perusio/nginx_ensite.git) along with the `/etc/nginx/sites-available` folder.

### Step 10 - Install Wordpress 

Grab the latest version of Wordpress and move it to your `/var/www` folder:

```
wget http://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz 
mkdir /var/www/wordpress
mv latest /var/www/wordpress/production
```

Configure Wordpress:

```
cd /var/www/wordpress/production
cp wp-config-sample.php wp-config.php
curl -s https://api.wordpress.org/secret-key/1.1/salt/ 
```

The last of those three commands will provide you with some secrets to add to the `wp-config.php` file. You also need to add your database credentials to this config file.

If you want Wordpress to be able to directly download plugins/updates you should also add:

```
define('FS_METHOD', 'direct');
```

Next you'll want to set up some file permissions:

```
mkdir /var/www/wordpress/production/wp-content/uploads
sudo chown -R www-data:www-data /var/www/winnipegelection/production/wp-content/ 
```

### Step 11 - Configure Nginx to host Wordpress

Here's a sample Nginx file for Wordpress hosting:

```
server {
    listen 80 default_server;
    listen [::]:80 default_server;

    root /var/www/wordpress/production;

    index index.php;

    server_name _;

    location = /favicon.ico {
        log_not_found off;
        access_log off;
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    # Required for supporting fancy permalinks
    location / {
        try_files $uri $uri/ /index.php?q=$uri&$args;
    }

    # pass PHP scripts to FastCGI server
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.2-fpm.sock;
    }
}
```

### Step 12 - Extra Security Stuff (Optional)

Enable SSL with certbot:

```
sudo apt-get update
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install python-certbot-nginx
sudo certbot --nginx
sudo service nginx reload
```

Certbox will also add a daily cron task to check and renew certificates if required.

Enable the [ufw](https://help.ubuntu.com/community/UFW) Firewall to only permit web (port 80) and ssh (port 22) traffic:

``` 
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https
sudo ufw enable
```

At this point you might also want to install [fail2ban](http://www.fail2ban.org/). I found the following two tutorials helpful:

- [How To Protect SSH with fail2ban on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04)
- [How to Secure an Nginx Server with Fail2Ban](http://snippets.aktagon.com/snippets/554-how-to-secure-an-nginx-server-with-fail2ban) (I only used the `badbots` and `noscript` jails.)

Be cautious with fail2ban. You don't want to lock yourself out of your own server. If you do get blocked, you can login using the Digital Ocean web console for your droplet.

### Step 12 - Test Your Sever

Congrats! At this point you should now be able to load either a Rails app or Wordpress. When testing your app be sure to trigger actions that both read and write to your database to ensure that MariaDB is properly configured.

### Step 14 - Keeping Your System Up To Date

When you connect via SSH to your droplet a login message will let you know if there are pending security updates. To download and install these updates, run the following commands:

``` 
sudo apt-get update
sudo apt-get upgrade
sudo apt-get dist-upgrade
```

After which you may have to restart your droplet:

``` 
sudo reboot
```

To list newly available Ruby versions:

``` 
rbenv install -l
```

To install a new version:

``` 
rbenv install <version number>
rbenv global <version number>
rbenv rehash
```

To install a new version of Rails, update your Rails project `Gemfile` and then run:

``` 
bundle update
rbenv rehash
```


