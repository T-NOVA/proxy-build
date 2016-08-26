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
  config.vm.box = "maier/alpine-3.4-x86_64"
  config.vm.host_name = "pxaas"

  # VirtualBox
  config.vm.provider "virtualbox" do |v|
    v.memory = 512
    v.cpus = 1
  end

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  config.vm.network "private_network", type: "dhcp"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"

  # Disable the default sharing
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # Configure local DNS resolution
  config.vm.provision "shell", inline: "echo '127.0.0.1 localhost pxaas' | tee /etc/hosts"
  config.vm.provision "shell", inline: "echo 'pxaas' | tee /etc/hostname"

  # Upgrade to edge to enable the testing repo (for squidguard)
  # https://wiki.alpinelinux.org/wiki/Upgrading_Alpine
  config.vm.provision "shell", inline: <<-SHELL
    sed -i -e 's/v3\.4/edge/g' /etc/apk/repositories
    sed -e 's/main/community/' /etc/apk/repositories | head -n 1 | tee -a /etc/apk/repositories
    sed -e 's/main/testing/' /etc/apk/repositories | head -n 1 | tee -a /etc/apk/repositories
    sync
  SHELL

  config.vm.provision "shell", inline: <<-SHELL
    apk upgrade -U --available
    cat /etc/*release*
    sync
  SHELL

  # Install random password generation
  config.vm.provision "shell", inline: "apk add pwgen"

  # Install and enable cloud-init
  config.vm.provision "shell", inline: <<-SHELL
    apk add cloud-init
    rc-update add cloud-init
    rc-update add cloud-config
    rc-update add cloud-final
  SHELL

  # Install and enable squid squidguard mariadb
  config.vm.provision "shell", inline: <<-SHELL
    apk add squid squidguard mariadb mariadb-client
    rc-update add mariadb
    rc-update add squid
  SHELL

  # Install and enable collectd and plugins
  config.vm.provision "shell", inline: <<-SHELL
    apk add collectd collectd-curl collectd-python collectd-write_http collectd-network collectd-mysql
    # apk add collectd-apache
    # apk add collectd-nginx
    # apk add collectd-rrdtool
    rc-update add collectd
  SHELL

  # Install and enable a web server with PHP
  config.vm.provision "shell", inline: <<-SHELL
    apk add php5 php5-cli php5-mysqli php5-imagick php5-zlib php5-fpm php5-mcrypt php5-gd php5-phar php5-curl php5-apache2 php5-intl php5-json php5-openssl php5-pdo_mysql php5-ctype
    rc-update add apache2
  SHELL

  # Install various tools
  config.vm.provision "shell", inline: "apk add git rsync py-pip"

  # TODO
  # Install and config the Alpine Configuration Framework (ACF)
  config.vm.provision "shell", inline: <<-SHELL
    apk add acf-squid
    setup-acf
    rc-update add mini_httpd
  SHELL

  # Configure MariaDB and add a random pwd for root
  config.vm.provision "shell", inline: <<-SHELL
    if [ ! -d "/run/mysqld" ]; then
      mkdir -p /run/mysqld
    fi
    chown -R mysql:mysql /run/mysqld

    if [ -d /var/lib/mysql/mysql ]; then
      echo "[i] MySQL directory already present, skipping creation"
      chown -R mysql:mysql /var/lib/mysql
    else
      echo "[i] MySQL data directory not found, creating initial DBs"

      chown -R mysql:mysql /var/lib/mysql

      mysql_install_db --user=mysql > /dev/null

      if [ "$MYSQL_ROOT_PASSWORD" = "" ]; then
        MYSQL_ROOT_PASSWORD=`pwgen 16 1`
        echo "[i] MySQL root Password: $MYSQL_ROOT_PASSWORD"
        echo $MYSQL_ROOT_PASSWORD > mysql_root_pass
      fi

      /usr/bin/mysqld --user=mysql --bootstrap --verbose=0 << EOF
USE mysql;
FLUSH PRIVILEGES;
CREATE USER 'root'@'%';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
UPDATE user SET password=PASSWORD("$MYSQL_ROOT_PASSWORD") WHERE user='root';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'127.0.0.1' WITH GRANT OPTION;
UPDATE user SET password=PASSWORD("") WHERE user='root' AND host='127.0.0.1';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' WITH GRANT OPTION;
UPDATE user SET password=PASSWORD("") WHERE user='root' AND host='localhost';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'pxaas' WITH GRANT OPTION;
UPDATE user SET password=PASSWORD("") WHERE user='root' AND host='pxaas';
FLUSH PRIVILEGES;
EOF
    fi
  SHELL

  # Create a new DB and user
  config.vm.provision "shell", inline: <<-SHELL
    rc-service mariadb start
    DASHBOARDUSER_PASS=`pwgen 16 1`
    echo $DASHBOARDUSER_PASS > dashboarduser_pass
    mysql -u root -h localhost << EOF
FLUSH PRIVILEGES;
USE mysql;
CREATE USER 'dashboarduser'@'localhost';
UPDATE user SET password=PASSWORD("$DASHBOARDUSER_PASS") WHERE user='dashboarduser';
CREATE DATABASE dashboarddb;
GRANT ALL PRIVILEGES ON dashboarddb.* TO dashboarduser@localhost;
FLUSH PRIVILEGES;
EOF
  SHELL

  # Copy squid and squidguard config to the box
  config.vm.provision "file", source: "./conf/squid.conf", destination: "squid.conf"
  config.vm.provision "file", source: "./conf/squidGuard.conf", destination: "squidGuard.conf"
  config.vm.provision "shell", inline: <<-SHELL
    DASHBOARDUSER_PASS=`cat dashboarduser_pass`
    sed -i "s/password 12345678/password $DASHBOARDUSER_PASS/g" squid.conf
    sed -i "s/mysqlpassword 12345678/mysqlpassword $DASHBOARDUSER_PASS/g" squidGuard.conf
    cp /etc/squid/squid.conf /etc/squid/squid.conf-dist
    cp /etc/squidGuard/squidGuard.conf /etc/squidGuard/squidGuard.conf-dist
    cp squid.conf /etc/squid/
    cp squidGuard.conf /etc/squidGuard/
  SHELL

  # TODO Run squidguard on the blacklists dir
  # Download the black/whitelists
  # See http://dsi.ut-capitole.fr/documentations/cache/squidguard_en.html
  # and http://dsi.ut-capitole.fr/blacklists/index_en.php
  config.vm.provision "shell", inline: <<-SHELL
    cd /etc/squidGuard
    mkdir /etc/squidGuard/blacklists && cd /etc/squidGuard/blacklists
    rsync -arpogvt rsync://ftp.ut-capitole.fr/blacklist/dest/ .
    wget -c http://www.squidblacklist.org/downloads/whitelist.txt
    chown -R squid:squid /etc/squidGuard/blacklists
    squidGuard -C all -b -d
    chown -R squid:squid /etc/squidGuard/blacklists
  SHELL

  # Clone the dashboard
  config.vm.provision "shell", privileged: "false", inline: <<-SHELL
    git clone https://github.com/T-NOVA/Squid-dashboard dashboard
    cd dashboard
    git checkout devel
    cd ..
    sudo chown -R vagrant:vagrant dashboard
  SHELL

  # Configure the dashboard
  config.vm.provision "shell", privileged: "false", inline: <<-SHELL
    DASHBOARDUSER_PASS=`cat dashboarduser_pass`
    sed -i "s/12345678/$DASHBOARDUSER_PASS/g" dashboard/config/db.php
    sudo chown -R vagrant:vagrant dashboard
  SHELL

  # Install Composer and plugins
  config.vm.provision "shell", inline: <<-SHELL
    cd /tmp
    curl -s http://getcomposer.org/installer | php
    mv composer.phar /usr/local/bin/composer
  SHELL
  config.vm.provision "shell", privileged: "false", inline: <<-SHELL
    composer global require "fxp/composer-asset-plugin:~1.2.1"
    cd dashboard
    composer install
    cd ..
    sudo chown -R vagrant:vagrant dashboard
  SHELL

  # Migrate DB to MySQL
  # Create user admin:administrator
  # Populate the DB with the blacklisted domains (this step may take a long time)
  config.vm.provision "shell", privileged: "false", inline: <<-SHELL
    cd dashboard
    php yii migrate/up --interactive=0 --migrationPath=@vendor/dektrium/yii2-user/migrations
    php yii createusers/create
    php yii migrate --interactive=0
  SHELL

  # Deploy the dashboard
  config.vm.provision "shell", inline: <<-SHELL
    ln -s /home/vagrant/dashboard /var/www/localhost/htdocs/
    chown -R apache:apache /var/www/localhost/htdocs/dashboard/web/assets
    chown -R apache:apache /var/www/localhost/htdocs/dashboard/runtime
  SHELL

  # Secure the apache httpd
  config.vm.provision "shell", inline: <<-SHELL
    sed -i 's/ServerTokens OS/ServerTokens Prod/g' /etc/apache2/httpd.conf
    sed -i 's/ServerAdmin you@example.com/ServerAdmin admin@t-nova.eu/g' /etc/apache2/httpd.conf
  SHELL

  # Apache rewrite module required for the dashboard
  config.vm.provision "shell", inline: <<-'SHELL'
    sed -i 's/#LoadModule rewrite_module modules\/mod_rewrite\.so/LoadModule rewrite_module modules\/mod_rewrite\.so/g' /etc/apache2/httpd.conf
  SHELL

  # Configure the httpd web root
  config.vm.provision "file", source: "./conf/dashboard-dir.conf", destination: "dashboard-dir.conf"
  config.vm.provision "shell", inline: <<-'SHELL'
    cp dashboard-dir.conf /etc/apache2/conf.d/
    sed -i 's/DocumentRoot[[:space:]]\+\"\/var\/www\/localhost\/htdocs\"$/DocumentRoot \"\/var\/www\/localhost\/htdocs\/dashboard\/web\"/g' /etc/apache2/httpd.conf
  SHELL

  # Enable service control for the dashboard
  config.vm.provision "shell", inline: <<-SHELL
    echo "apache ALL=(ALL) NOPASSWD:ALL" | tee /etc/sudoers.d/apache
    chmod 440 /etc/sudoers.d/apache
  SHELL


  # TODO
  # Scaling with haproxy

  # TODO
  # Test squidanalyzer

  # TODO
  # Test https://wiki.alpinelinux.org/wiki/CGP


  # ALWAYS RUN
  # Fix permissions of conf files
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    chown apache:apache /etc/squid/squid.conf
    chown apache:apache /etc/squidGuard/squidGuard.conf
  SHELL

  # ALWAYS RUN
  # Pull new code into the dashboard
  config.vm.provision "shell", privileged: "false", run: "always", inline: <<-SHELL
    cd dashboard
    git pull
    cd ..
    chown -R vagrant:vagrant dashboard
    chown -R apache:apache /var/www/localhost/htdocs/dashboard/web/assets
    chown -R apache:apache /var/www/localhost/htdocs/dashboard/runtime
  SHELL

  # ALWAYS RUN
  # Clear /etc/network/interfaces and reload
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    cat << EOF > /etc/network/interfaces
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp
    hostname pxaas

EOF
  SHELL

  # ALWAYS RUN
  # Clean the apk cache and reload
  config.vm.provision "shell", run: "always", inline: "rm -rf /var/cache/apk/*"
  config.vm.provision :reload

  # ALWAYS RUN
  # Check runtime services are up
  config.vm.provision "shell", run: "always", inline: "rc-status -a"

end

