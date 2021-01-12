# LAMP Server Setup for Drupal

This guide assumes you're running an Ubuntu 20.04 VM in VirtualBox, using the included Vagrantfile. Install Vagrant and VirtualBox, then run `vagrant up` in this directory.

Run `vagrant ssh` to log into the VM, then run all the commands below:

```
# Update apt caches.
sudo apt update

# Install MySQL (or in this case, MariaDB) and unzip packages.
sudo apt install -y mariadb-server mariadb-client unzip

# Run through the installation process.
sudo mysql_secure_installation

# Create a Drupal database while logged into the MySQL cli.
sudo mysql -u root

CREATE DATABASE drupal;
GRANT ALL ON drupal.* TO 'drupal'@'localhost' IDENTIFIED BY 'mypassword';
FLUSH PRIVILEGES;
\q

# Install PHP.
sudo apt install -y php php-{cli,fpm,json,common,mysql,zip,gd,intl,mbstring,curl,xml,pear,tidy,soap,bcmath,xmlrpc}

# Install Apache with mod_php.
sudo apt install -y apache2 libapache2-mod-php

# Configure a couple important PHP settings.
sudo nano /etc/php/7.4/apache2/php.ini
```

Find the following lines and change them to these settings and save the file:

```
memory_limit = 512M
date.timezone = America/Chicago
```

And back to running commands:

```
# Configure a Drupal virtual host for Apache.
sudo nano /etc/apache2/sites-available/drupal.conf
```

```
<VirtualHost *:80>
     ServerName example.com
     ServerAlias www.example.com
     ServerAdmin webmaster@example.com
     DocumentRoot /var/www/html/drupal/web

     CustomLog ${APACHE_LOG_DIR}/access.log combined
     ErrorLog ${APACHE_LOG_DIR}/error.log

      <Directory /var/www/html/drupal/web>
            Options Indexes FollowSymLinks
            AllowOverride All
            Require all granted
            RewriteEngine on
            RewriteBase /
            RewriteCond %{REQUEST_FILENAME} !-f
            RewriteCond %{REQUEST_FILENAME} !-d
            RewriteRule ^(.*)$ index.php?q=$1 [L,QSA]
   </Directory>
</VirtualHost>
```

(We'll finish setting up Apache later, after we have Drupal installed.)

```
# Install Composer (https://getcomposer.org/download/)
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
sudo mv composer.phar /usr/local/bin/composer

# Prepare the /var/www directory.
sudo chown -R www-data:www-data /var/www

# Create a Drupal project codebase with Composer as the Apache user.
sudo su -l www-data -s /bin/bash
composer create-project drupal/recommended-project /var/www/html/drupal
exit

# Finish configuring Apache and restart it to pick up the new site.
sudo a2enmod rewrite
sudo a2ensite drupal.conf
sudo a2dissite 000-default.conf
sudo systemctl restart apache2
```

Visit the site in your browser: http://192.168.80.80/

Install Drupal (DB name: `drupal`, DB user: `drupal`, DB password: `mypassword`), and celebrate!
