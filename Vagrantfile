# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure(2) do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://atlas.hashicorp.com/search.
  config.vm.box = "hashicorp/precise64"
  config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"
#  config.vm.provision :shell, path: "bootstrap.sh"

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  config.vm.network "forwarded_port", guest: 80, host: 8090

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  config.vm.network "private_network", ip: "192.168.33.10"
  config.vm.hostname = "dev.vagrant.com"
  #config.hostsupdater.aliases = ["dev."]
  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  config.vm.synced_folder "docroot/", "/vagrant/public"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  # config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end
  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Define a Vagrant Push strategy for pushing to Atlas. Other push strategies
  # such as FTP and Heroku are also available. See the documentation at
  # https://docs.vagrantup.com/v2/push/atlas.html for more information.
  # config.push.define "atlas" do |push|
  #   push.app = "YOUR_ATLAS_USERNAME/YOUR_APPLICATION_NAME"
  # end

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  config.vm.provision "shell", inline: <<-SHELL
    #! /usr/bin/env bash

    # Variables
    APPENV=local
    DBHOST=localhost
    DBNAME=dbname
    DBUSER=dbuser
    DBPASSWD=test123

    echo -e "\n--- Mkay, installing now... ---\n"

    echo -e "\n--- Updating packages list ---\n"
    apt-get -qq update

    echo -e "\n--- Install base packages ---\n"
    apt-get -y install vim curl build-essential python-software-properties git > /dev/null 2>&1

    echo -e "\n--- Add some repos to update our distro ---\n"
    add-apt-repository ppa:ondrej/php5 > /dev/null 2>&1
    add-apt-repository ppa:chris-lea/node.js > /dev/null 2>&1

    echo -e "\n--- Updating packages list ---\n"
    apt-get -qq update

    echo -e "\n--- Install MySQL specific packages and settings ---\n"
    echo "mysql-server mysql-server/root_password password $DBPASSWD" | debconf-set-selections
    echo "mysql-server mysql-server/root_password_again password $DBPASSWD" | debconf-set-selections
    echo "phpmyadmin phpmyadmin/dbconfig-install boolean true" | debconf-set-selections
    echo "phpmyadmin phpmyadmin/app-password-confirm password $DBPASSWD" | debconf-set-selections
    echo "phpmyadmin phpmyadmin/mysql/admin-pass password $DBPASSWD" | debconf-set-selections
    echo "phpmyadmin phpmyadmin/mysql/app-pass password $DBPASSWD" | debconf-set-selections
    echo "phpmyadmin phpmyadmin/reconfigure-webserver multiselect none" | debconf-set-selections
    apt-get -y install mysql-server-5.5 phpmyadmin > /dev/null 2>&1

    echo -e "\n--- Setting up our MySQL user and db ---\n"
    mysql -uroot -p$DBPASSWD -e "CREATE DATABASE $DBNAME"
    mysql -uroot -p$DBPASSWD -e "grant all privileges on $DBNAME.* to '$DBUSER'@'localhost' identified by '$DBPASSWD'"

    echo -e "\n--- Installing PHP-specific packages ---\n"
    apt-get -y install php5 apache2 libapache2-mod-php5 php5-curl php5-gd php5-mcrypt php5-mysql php-apc > /dev/null 2>&1

    echo -e "\n--- Enabling mod-rewrite ---\n"
    a2enmod rewrite > /dev/null 2>&1

    echo -e "\n--- Allowing Apache override to all ---\n"
    sed -i "s/AllowOverride None/AllowOverride All/g" /etc/apache2/apache2.conf

    echo -e "\n--- Setting document root to public directory ---\n"
    rm -rf /var/www
    ln -fs /vagrant/public /var/www

    echo -e "\n--- We definitly need to see the PHP errors, turning them on ---\n"
    sed -i "s/error_reporting = .*/error_reporting = E_ALL/" /etc/php5/apache2/php.ini
    sed -i "s/display_errors = .*/display_errors = On/" /etc/php5/apache2/php.ini

    echo -e "\n--- Turn off disabled pcntl functions so we can use Boris ---\n"
    sed -i "s/disable_functions = .*//" /etc/php5/cli/php.ini

    echo -e "\n--- Configure Apache to use phpmyadmin ---\n"
    echo -e "\n\nListen 81\n" >> /etc/apache2/ports.conf
    cat > /etc/apache2/conf-available/phpmyadmin.conf << "EOF"
<VirtualHost *:81>
    ServerAdmin webmaster@localhost
    DocumentRoot /usr/share/phpmyadmin
    DirectoryIndex index.php
    ErrorLog ${APACHE_LOG_DIR}/phpmyadmin-error.log
    CustomLog ${APACHE_LOG_DIR}/phpmyadmin-access.log combined
</VirtualHost>
EOF
    a2enconf phpmyadmin > /dev/null 2>&1

    echo -e "\n--- Add environment variables to Apache ---\n"
    cat > /etc/apache2/sites-enabled/000-default.conf <<EOF
<VirtualHost *:80>
    DocumentRoot /var/www
    ErrorLog \${APACHE_LOG_DIR}/error.log
    CustomLog \${APACHE_LOG_DIR}/access.log combined
    SetEnv APP_ENV $APPENV
    SetEnv DB_HOST $DBHOST
    SetEnv DB_NAME $DBNAME
    SetEnv DB_USER $DBUSER
    SetEnv DB_PASS $DBPASSWD
</VirtualHost>
EOF

    echo -e "\n--- Restarting Apache ---\n"
    service apache2 restart > /dev/null 2>&1

    echo -e "\n--- Installing Composer for PHP package management ---\n"
    curl --silent https://getcomposer.org/installer | php > /dev/null 2>&1
    mv composer.phar /usr/local/bin/composer

    echo -e "\n--- Installing NodeJS and NPM ---\n"
    apt-get -y install nodejs > /dev/null 2>&1
    curl --silent https://npmjs.org/install.sh | sh > /dev/null 2>&1

    echo -e "\n--- Installing javascript components ---\n"
    npm install -g gulp bower > /dev/null 2>&1

    echo -e "\n--- Updating project components and pulling latest versions ---\n"
    cd /vagrant
    sudo -u vagrant -H sh -c "composer install" > /dev/null 2>&1
    cd /vagrant/client
    sudo -u vagrant -H sh -c "npm install" > /dev/null 2>&1
    sudo -u vagrant -H sh -c "bower install -s" > /dev/null 2>&1
    sudo -u vagrant -H sh -c "gulp" > /dev/null 2>&1

    echo -e "\n--- Creating a symlink for future phpunit use ---\n"
    ln -fs /vagrant/vendor/bin/phpunit /usr/local/bin/phpunit

    echo -e "\n--- Add environment variables locally for artisan ---\n"
    cat >> /home/vagrant/.bashrc <<EOF
# Set envvars
export APP_ENV=$APPENV
export DB_HOST=$DBHOST
export DB_NAME=$DBNAME
export DB_USER=$DBUSER
export DB_PASS=$DBPASSWD
EOF

    ## -- Percona Install --
    ## msmith: Left out for now.
    #wget https://repo.percona.com/apt/percona-release_0.1-3.$(lsb_release -sc)_all.deb
    #dpkg -i percona-release_0.1-3.$(lsb_release -sc)_all.deb
    #apt-get update
    #apt-get install percona-server-server-5.5 -y
  SHELL
end
