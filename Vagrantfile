# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|

  config.vm.box = "bento/ubuntu-16.04"
  config.vm.host_name = "pxaas"

  # VirtualBox provider
  config.vm.provider "virtualbox" do |v|
    v.memory = 512
    v.cpus = 1
  end

  # Networks
  config.vm.network "private_network", type: "dhcp" # monitoring plane
  config.vm.network "private_network", type: "dhcp" # management plane
  config.vm.network "public_network" # data plane
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Disable default sharing
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # Provisioning

  # Fix the no tty error (see https://github.com/Varying-Vagrant-Vagrants/VVV/issues/517)
  config.vm.provision "fix-no-tty", type: "shell" do |s|
    s.privileged = false
    s.inline = "sudo sed -i '/tty/!s/mesg n/tty -s \\&\\& mesg n/' /root/.profile"
  end

  # Update apt repos
  config.vm.provision "shell", inline: "DEBIAN_FRONTEND=noninteractive apt-get update"

  # Install system software
  config.vm.provision "shell", inline: <<-SHELL
    DEBIAN_FRONTEND=noninteractive apt-get install -y git curl python-pip htop pwgen debconf-utils zerofree
  SHELL

  # Setup cloud-init and collectd
  config.vm.provision "shell", inline: <<-SHELL
    DATASOURCE_LIST="ConfigDrive, Openstack, None"
    debconf-set-selections <<< "cloud-init cloud-init/datasources multiselect ${DATASOURCE_LIST}"
    debconf-set-selections <<< "unattended-upgrades unattended-upgrades/enable_auto_updates boolean true"
    DEBIAN_FRONTEND=noninteractive apt-get install -y collectd unattended-upgrades collectd-utils cloud-init cloud-initramfs-growroot
  SHELL

  # Generate random passwords for MySQL root, dashboarduser, dashboard cookie, dashboard admin, vagrant user
  config.vm.provision "shell", inline: <<-SHELL
    echo `pwgen 16 1` > mysql_root_pass
    echo `pwgen 16 1` > mysql_dashboarduser_pass
    echo `pwgen 16 1` > dashboard_cookie
    echo `pwgen 8 1` > dashboard_admin
    echo `pwgen 8 1` > user_vagrant
    chmod 600 mysql_root_pass mysql_dashboarduser_pass dashboard_cookie dashboard_admin user_vagrant
  SHELL

  # Install squid, squidclient and squidguard
  config.vm.provision "shell", inline: <<-SHELL
    debconf-set-selections <<< "squid squid/fix_cachedir_perms boolean true"
    DEBIAN_FRONTEND=noninteractive apt-get install -y squid squidclient squidguard
  SHELL

  # Install LAMP components
  config.vm.provision "shell", inline: <<-SHELL
    MYSQL_ROOT_PASS=`cat mysql_root_pass`
    debconf-set-selections <<< "maria-db-10.0 mysql-server/root_password password ${MYSQL_ROOT_PASS}"
    debconf-set-selections <<< "maria-db-10.0 mysql-server/root_password seen true"
    debconf-set-selections <<< "maria-db-10.0 mysql-server/root_password_again password ${MYSQL_ROOT_PASS}"
    debconf-set-selections <<< "maria-db-10.0 mysql-server/root_password_again seen true"
    debconf-set-selections <<< "maria-db-10.0 mariadb-server-10.0/postrm_remove_databases boolean true"
    DEBIAN_FRONTEND=noninteractive apt-get install -y apache2 mariadb-server php php-curl php-intl php-mcrypt php-mysql php-imagick php-zip php-gd php-json php-mbstring

    # mysql_secure_installation
    systemctl start mysql.service
    mysqladmin -u root password "${MYSQL_ROOT_PASS}"
    mysqladmin -u root -h localhost password "${MYSQL_ROOT_PASS}"
  SHELL

  # Create a new DB and user
  config.vm.provision "shell", inline: <<-SHELL
    MYSQL_ROOT_PASS=`cat mysql_root_pass`
    MYSQL_DASHBOARDUSER_PASS=`cat mysql_dashboarduser_pass`
    cat <<EOF | mysql -u root -p"${MYSQL_ROOT_PASS}"
CREATE DATABASE dashboarddb;
CREATE USER 'dashboarduser'@'localhost' IDENTIFIED BY "${MYSQL_DASHBOARDUSER_PASS}";
GRANT ALL PRIVILEGES ON dashboarddb.* TO dashboarduser@localhost;
EOF
  SHELL

  # Copy squid and squidguard config to the box
  config.vm.provision "file", source: "./conf/squid.conf", destination: "squid.conf"
  config.vm.provision "file", source: "./conf/squidGuard.conf", destination: "squidGuard.conf"
  config.vm.provision "shell", inline: <<-SHELL
    cp squid.conf /etc/squid
    cp squidGuard.conf /etc/squidguard
    chown -R proxy:proxy /etc/squid*
  SHELL

  # Download the black/whitelists
  # See http://dsi.ut-capitole.fr/documentations/cache/squidguard_en.html
  # and http://dsi.ut-capitole.fr/blacklists/index_en.php
  config.vm.provision "shell", inline: <<-SHELL
    cd /var/lib/squidguard
    rsync -arpogvt rsync://ftp.ut-capitole.fr/blacklist .
    mv dest/* /var/lib/squidguard/db
    wget -c http://www.squidblacklist.org/downloads/whitelist.txt -o db/whitelist
    squidGuard -d -C all
  SHELL

  # Clone the dashboard
  config.vm.provision "shell", inline: <<-SHELL
    su vagrant -l -c "git clone https://github.com/T-NOVA/Squid-dashboard dashboard"
    su vagrant -l -c "cd dashboard; git checkout devel"
  SHELL

  # Configure the dashboard
  config.vm.provision "shell", inline: <<-SHELL
    DASHBOARDUSER_PASS=`cat mysql_dashboarduser_pass`
    sed -i "s/12345678/$DASHBOARDUSER_PASS/g" dashboard/config/db.php
  SHELL

  # Install Composer and plugins
  config.vm.provision "shell", inline: <<-SHELL
    cd /tmp
    curl -sS http://getcomposer.org/installer | php
    mv composer.phar /usr/local/bin/composer
    su vagrant -l -c "cd dashboard; composer global require 'fxp/composer-asset-plugin:^1.2.0'"
    su vagrant -l -c "cd dashboard; composer install"
  SHELL

  # Migrate DB to MySQL
  # Create user admin:adminadmin
  # Populate the DB with the blacklisted domains (this step may take a long time)
  config.vm.provision "shell", inline: <<-SHELL
    su vagrant -l -c "cd dashboard; php yii migrate/up --interactive=0 --migrationPath=@vendor/dektrium/yii2-user/migrations"
    su vagrant -l -c "cd dashboard; php yii createusers/create"
    su vagrant -l -c "cd dashboard; php yii migrate --interactive=0"
  SHELL

  # Deploy the dashboard
  config.vm.provision "shell", inline: <<-SHELL
    ln -s /home/vagrant/dashboard /var/www/html/
    chown -R www-data:www-data /var/www/html/dashboard/web/assets
    chown -R www-data:www-data /var/www/html/dashboard/runtime
  SHELL

  # Apache rewrite module required for the dashboard
  config.vm.provision "shell", inline: <<-SHELL
    /usr/sbin/a2enmod rewrite
    systemctl restart apache2.service
  SHELL

  # Configure the apache web root
  config.vm.provision "file", source: "./conf/dashboard-dir.conf", destination: "dashboard-dir.conf"
  config.vm.provision "shell", inline: <<-'SHELL'
    cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/000-default.conf.dist
    cp /home/vagrant/dashboard-dir.conf /etc/apache2/conf-available/
    chown root:root /etc/apache2/conf-available/dashboard-dir.conf
    ln -sf /etc/apache2/conf-available/dashboard-dir.conf /etc/apache2/conf-enabled/dashboard-dir.conf
    ln -sf /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-enabled/000-default.conf
    sed -i 's/DocumentRoot[[:space:]]\+\/var\/www\/html$/DocumentRoot \/var\/www\/html\/dashboard\/web/g' /etc/apache2/sites-available/000-default.conf
  SHELL

  # Enable service control from the dashboard
  config.vm.provision "shell", inline: <<-SHELL
    echo "www-data ALL=(ALL) NOPASSWD:ALL" | tee /etc/sudoers.d/www-data
    chmod 440 /etc/sudoers.d/www-data
  SHELL

  # ALWAYS RUN
  # Fix permissions on conf files
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    chown www-data:www-data /etc/squid/squid.conf
    chown www-data:www-data /etc/squidguard/squidGuard.conf
    chmod g+rw /etc/squid/squid.conf
    chmod g+rw /etc/squidguard/squidGuard.conf
  SHELL

  # Configure services
  config.vm.provision "shell", inline: <<-SHELL
    systemctl enable mysql.service
    systemctl restart mysql.service

    systemctl enable apache2.service
    systemctl restart apache2.service

    systemctl enable squid.service
    systemctl restart squid.service

    systemctl enable cloud-config.service
    systemctl enable cloud-init.service
    systemctl enable cloud-final.service
    systemctl restart cloud-config.service
    systemctl restart cloud-init.service
    systemctl restart cloud-final.service
  SHELL

  # ALWAYS RUN
  # Reload the dashboard
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    su vagrant -l -c "cd dashboard; git stash; git pull; git stash apply; git stash drop"
  SHELL

  # Stop all services so that the FS can be mounted read-only
  config.vm.provision "shell", inline: <<-SHELL
    systemctl stop cloud-init.service
    systemctl stop cloud-final.service
    systemctl stop cloud-config.service
    systemctl stop squid.service
    systemctl stop apache2.service
    systemctl stop mysql.service
    fuser -v -m / # services with open files
  SHELL

  # Purge unneeded files and zero-out free space
  config.vm.provision "shell", inline: <<-SHELL
    DEBIAN_FRONTEND=noninteractive apt-get remove -y --purge debconf-utils pwgen
    rm -rf /var/cache/apt
    # Use zerofree instead of dd
    # dd if=/dev/zero of=/tmp/zeros bs=1M
    # rm /tmp/zeros
    mount -o remount,ro /
    zerofree /
  SHELL

end

