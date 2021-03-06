---
layout: page
title: VPS Ruby on Rails Hosting Revisted
---

*ODM Technology Post*

**In this post we will configure Ruby on Rails on a [Digital Ocean](https://m.do.co/c/9c57a647fd20) VPS.**

**This is an update to [a similar post from two years ago](/2014/04/07/vps-ruby-on-rails-hosting/).** This article was originally posted in January of 2015, but was updated again in November of 2016.

Our final hosting setup will be:

- OS: Linux - [Ubuntu](http://ubuntu.com/) LTS Version
- Web Server: [Nginx](http://nginx.org/)
- Rails Hosting: [Phussion Passenger](https://www.phusionpassenger.com/)
- Database Server: [PostgreSQL](http://www.postgresql.org/)
- Language and Framework: [Ruby](http://www.ruby-lang.org/) 2.x & [Rails](http://rubyonrails.org/) 5.x

*This tutorial assumes a familiarity with the Linux command line, server administration, and Rails configuration.*

### Background Information

A VPS is a [Virtual Private Server](https://en.wikipedia.org/wiki/Virtual_private_server). In our case, a virtualized Ubuntu Linux server from [Digital Ocean](https://m.do.co/c/9c57a647fd20) to host [ODM](http://opendemocracymanitoba.ca) Ruby projects.

The 512MB plan on [Digital Ocean](https://m.do.co/c/9c57a647fd20) is only $5 a month. Nice.

### Automation and Config Management

This post does not rely on a configuration management tool.

I figured I'd be done setting up the server long before I'd figured out the intricacies of [Puppet](http://puppetlabs.com/), [Chef](http://www.getchef.com/), or [Ansible](http://www.ansible.com). Considering all the changes when compared with [the install I did in 2014](/2014/04/07/vps-ruby-on-rails-hosting/) I'm glad I hadn't automated the process yet.

That said, if you can see yourself setting up multiple servers following this tutorial, you may wish to look into one of the automated provisioning solution listed in the previous paragraph.

Also, if you only plan on hosting one Rails app on your VPS, [Digital Ocean's one-click Rails install](https://www.digitalocean.com/community/tutorials/how-to-use-the-ruby-on-rails-one-click-application-on-digitalocean) might be the easiest solution. Their one-click install uses [RVM](http://rvm.beginrescueend.com/) instead of rbenv and [Unicorn](http://unicorn.bogomips.org/) in place of Passenger. Although, fair warning, it requires much more than just one-click. 

### Step 1 - Create the Droplet

Digital Ocean (DO) calls their VPS instances Droplets. Once you have a DO account you can create new droplets by clicking on the 'Create' button in your admin dashboard. I selected the following options:

- Size: 1GB
- Region: Toronto 1
- Linux Distribution: Ubuntu 16.04 LTS

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

Some of the following steps require compilation, so it's nice to have a swap file on the server. By default DO droplets to not have swap files, but [adding one isn't difficult](https://www.digitalocean.com/community/tutorials/how-to-add-swap-on-ubuntu-14-04). Feel free to skip this step.

I created a 1GB swapfile:

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

Using swap too aggressively on a SSD drive can lead to hardware degradation. So let's dial down the "[swappiness](https://help.ubuntu.com/community/SwapFaq#What_is_swappiness_and_how_do_I_change_it.3F)" from 60 to 10:

``` 
echo 10 | sudo tee /proc/sys/vm/swappiness
echo vm.swappiness = 10 | sudo tee -a /etc/sysctl.conf
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
curl -sL https://deb.nodesource.com/setup_6.x | sudo -E bash 
sudo apt-get install git g++ make nodejs libsqlite3-dev postgresql postgresql-contrib postgresql-server-dev-9.5 gconf2 libcurl4-openssl-dev imagemagick zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev sqlite3 libxml2-dev libxslt1-dev python-software-properties libffi-dev
```

(The first line enables the installation of node.js for CoffeeScript support in Rails.)

### Step 5 - PostgreSQL Setup

Switch to the postgres user:

``` 
sudo su - postgres
```

Add a database role/user with a password:

``` 
createuser --pwprompt myapp_postgres_user
```

You're going to need the role/password details again in step 8.

Add a new database where `myapp` is the name of the Rails app you will be deploying:

``` 
createdb --owner myapp_postgres_user myapp_development
createdb --owner myapp_postgres_user myapp_test
createdb --owner myapp_postgres_user myapp_production
```

Switch back to the `serveruser`:

``` 
exit
```

### Step 6 - Install Ruby and Rails

We are going to install Ruby by way of [rbenv](https://github.com/sstephenson/rbenv).

``` 
git clone git://github.com/sstephenson/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
echo 'eval "$(rbenv init -)"' >> ~/.bash_profile
git clone git://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bash_profile
```

This command will provide you with lines to add to your .bash_profile. Add those lines and reload the file:

``` 
source ~/.bash_profile
```

Install Ruby (the latest version at the time of this post was 2.3.0):

``` 
rbenv install 2.3.3
rbenv global 2.3.3
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

### Step 7 - Install Passenger & Nginx:

Install [Phusion Passenger](https://www.phusionpassenger.com/) and the [Nginx](http://nginx.org/) web server:

``` 
# Install our PGP key and add HTTPS support for APT
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 561F9B9CAC40B2F7
sudo apt-get install -y apt-transport-https ca-certificates

# Add our APT repository
sudo sh -c 'echo deb https://oss-binaries.phusionpassenger.com/apt/passenger xenial main > /etc/apt/sources.list.d/passenger.list'
sudo apt-get update

# Install Passenger + Nginx
sudo apt-get install -y nginx-extras passenger
```

Enable Passenger in the Nginx config file:

``` 
sudo vim /etc/nginx/nginx.conf
```

And uncomment (remove the #) from the following lines:

``` 
# include /etc/nginx/passenger.conf;
```

Re-start Nginx:

``` 
sudo service nginx restart
```

### Step 8 - Install Your Rails App

Create the /var/www folder and give it permissions. If /var/www already exists you can skip the `mkdir` step.

``` 
sudo groupadd www-pub
sudo usermod -a -G www-pub serveruser 
sudo mkdir /var/www
sudo chown -R serveruser:www-pub /var/www
sudo chmod 2775 /var/www
```

And then place your Rails app in the `/var/www` folder. I did so by cloning the github repo.

Set up a ssh key for the server:

``` 
ssh-keygen -t rsa -b 4096 -C "stungeye@gmail.com"
```

Configure git:

``` 
git config --global user.email "youremail@example.com"
git config --global user.name "Your Name"
```

Clone the repository:

``` 
git clone git@github.com:OpenDemocracyManitoba/winnipegelection.git
```

Not shown in this tutorial: [Linking the SSH key created above to your Github account](https://help.github.com/articles/generating-ssh-keys#platform-linux).

### Step 9 - Configure Your App for Postgres

Remove sqlite from the Gemfile and replace it with the pg gem. And then update:

``` 
bundle update
```

Edit the `config/database.yml` [for use with Postgres](http://guides.rubyonrails.org/configuring.html#configuring-a-database) using the role/password you created in step 5. A sample production config might look like this:

```
production:
  adapter: postgresql
  encoding: unicode
  host: localhost
  pool: 5
  database: myapp_production
  username: myapp_postgres_user
  password: thepasswordyoupickedinstep5
```

Don't forget to run your migration!

### Step 10 - Configure Nginx to Host Your Rails App

To test your app, here's simple ngnix server block that you can put in your `/etc/nginx/sites-available/default` config file. You will need to edit this file using `sudo` and you should replace the existing server block that is already present in the file.

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

You can also set your `rails_env` to `production` if this is meant to be a production deployment. Doing so means you need to perform one extra step, described below as step 11.

Use [sites-available / sites-enabled config files](https://www.linode.com/docs/websites/nginx/how-to-configure-nginx#server-virtual-domains-configuration) if you want to host multiple projects on this server.

### Step 11 - Configure Rails for Production

When Rails runs in production mode there is an extra configuration step that you must before. You can skip this step if you set your `rails_env` to `development` in the last step.

In the `config` folder of your Rails Project you will find a file named `secrets.yml`. When running in production mode you will need to have a production `secret_key_base` set in this file. By default the file is configured to look for this secret in an OS environment variable called `SECRET_KEY_BASE`. You can also hardcode a secret into this file, but if you do, **be sure to remove the `secrets.yml` from your git repo**. You do not want to accidentally expose this secret to the world by pushing your project to a public Github repo. Rails uses this secret key to do things like encrypt session cookies, so exposing this secret would be a large security vulnerability.

Secrets can be generated from the command line by running the following command from your project folder:

```
rake secret
```

### Step 12 - Extra Security Stuff (Optional)

Enable the [ufw](https://help.ubuntu.com/community/UFW) Firewall to only permit web (port 80) and ssh (port 22) traffic:

``` 
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow www
sudo ufw enable
```

At this point you might also want to install [fail2ban](http://www.fail2ban.org/). I found the following two tutorials helpful:

- [How To Protect SSH with fail2ban on Ubuntu 14.04](https://www.digitalocean.com/community/tutorials/how-to-protect-ssh-with-fail2ban-on-ubuntu-14-04)
- [How to Secure an Nginx Server with Fail2Ban](http://snippets.aktagon.com/snippets/554-how-to-secure-an-nginx-server-with-fail2ban) (I only used the `badbots` and `noscript` jails.)

Be cautious with fail2ban. You don't want to lock yourself out of your own server. If you do get blocked, you can login using the Digital Ocean web console for your droplet.

### Step 13 - Test Your Sever

Congrats! At this point you should be able to load your Rails app via a web browser. When testing your app be sure to trigger actions that both read and write to your database to ensure that Postgres is properly configured.

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

To keep rbenv up to date:

``` 
rbenv update
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


