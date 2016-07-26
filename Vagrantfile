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
  config.vm.box = "debian/jessie64"
  config.vm.host_name = "pxaas"

  # VirtualBox
  config.vm.provider "virtualbox" do |v|
    v.memory = 1024
    v.cpus = 2
  end

  # KVM
  config.vm.provider "libvirt" do |v|
    # v.host = "localhost"
    # v.username = ""
    # v.password = ""
    # v.connect_via_ssh =
    v.memory = 1024
    v.cpus = 1
    # v.nested = "false"
    # v.graphics_type = "spice"
    # v.channel :type => "spicevmc", :target_name => "com.redhat.spice.0", :target_type => "virtio"
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
  config.vm.network "private_network", ip: "192.168.64.120"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"

  # Configure Shared folders. This provisions the VM's /home/proxyvnf/dashboard folder
  # to the host's ./shared folder (relative to current path)
  # config.vm.synced_folder "./shared", "/home/proxyvnf/dashboard",
  #   owner: "proxyvnf",
  #   group: "proxyvnf",
  #   mount_options: ["dmode=755,fmode=644"]

  # Disable the default sharing
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # Define a Vagrant Push strategy for pushing to Atlas. Other push strategies
  # such as FTP and Heroku are also available. See the documentation at
  # https://docs.vagrantup.com/v2/push/atlas.html for more information.
  # config.push.define "atlas" do |push|
  #   push.app = "YOUR_ATLAS_USERNAME/YOUR_APPLICATION_NAME"
  # end

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.

  # Fix the no tty error (see https://github.com/Varying-Vagrant-Vagrants/VVV/issues/517)
   config.vm.provision "fix-no-tty", type: "shell" do |s|
     s.privileged = false
     s.inline = "sudo sed -i '/tty/!s/mesg n/tty -s \\&\\& mesg n/' /root/.profile"
   end

  # Update apt repos
  config.vm.provision "shell", inline: "apt-get update"
  # puppet
  config.vm.provision "shell", inline: "apt-get install -y puppet"
  # ansible does not need a client, but needs python-apt to install packages
  config.vm.provision "shell", inline: "apt-get install -y python-apt"

  # Create and configure the user who runs the dashboard
  config.vm.provision "shell", inline: <<-SHELL
    useradd -m -U -G sudo proxyvnf
    sudo -u proxyvnf ssh-keygen -t rsa -q -N "" -f /home/proxyvnf/.ssh/id_rsa
  SHELL

  # Install LAMP components
  config.vm.provision "shell", inline: <<-SHELL
    # export DEBIAN_FRONTEND=noninteractive
    debconf-set-selections <<< "maria-db-10.0 mysql-server/root_password password 12345678"
    debconf-set-selections <<< "maria-db-10.0 mysql-server/root_password_again password 12345678"
    apt-get install -y apache2 mariadb-server php5 php5-curl php5-intl php5-mcrypt php5-mysql php5-imagick
    systemctl enable mysql.service
    systemctl restart mysql.service
    systemctl enable apache2.service
    systemctl restart apache2.service
  SHELL

  # Install squid and squidguard
  config.vm.provision "shell", inline: <<-SHELL
    apt-get install -y squidguard
    systemctl enable squid3.service
  SHELL

  # Install other software components
  config.vm.provision "shell", inline: <<-SHELL
    apt-get install -y git vim curl python-pip
  SHELL

  # Create a new DB and user
  config.vm.provision "shell", inline: <<-SHELL
    echo "CREATE DATABASE dashboarddb;
    CREATE USER 'dashboarduser'@'localhost' IDENTIFIED BY '12345678';
    GRANT ALL PRIVILEGES ON dashboarddb.* TO dashboarduser@localhost;" | mysql -u root -p12345678
  SHELL

  # Copy squid and squidguard config to the box
  config.vm.provision "file", source: "./conf/squid.conf", destination: "squid.conf"
  config.vm.provision "file", source: "./conf/squidguard.conf", destination: "squidguard.conf"
  config.vm.provision "shell", inline: <<-SHELL
    cp /home/vagrant/squid.conf /etc/squid3
    cp /home/vagrant/squidguard.conf /etc/squidguard
    chown -R proxy:proxy /etc/squid*
  SHELL

  # Download the black/whitelists
  # See http://dsi.ut-capitole.fr/documentations/cache/squidguard_en.html
  # ans http://dsi.ut-capitole.fr/blacklists/index_en.php
  config.vm.provision "shell", inline: <<-SHELL
    cd /home/proxyvnf
    sudo -u proxyvnf mkdir blacklists && cd blacklists
    sudo -u proxyvnf rsync -arpogvt rsync://ftp.ut-capitole.fr/blacklist .
    sudo -u proxy mkdir /etc/squidguard/blacklists
    sudo -u proxy cp -r dest/* /etc/squidguard/blacklists
    sudo -u proxyvnf wget -c http://www.squidblacklist.org/downloads/whitelist.txt
  SHELL

  # Clone the dashboard
  config.vm.provision "shell", inline: <<-SHELL
    sudo -u proxyvnf git clone https://github.com/T-NOVA/Squid-dashboard /home/proxyvnf/dashboard
  SHELL

  # Configure the dashboard
  config.vm.provision "shell", inline: <<-SHELL
    sed -e 's/primetel/12345678/g' /home/proxyvnf/dashboard/config/db.php > /home/proxyvnf/dashboard/config/db1.php
    mv /home/proxyvnf/dashboard/config/db1.php /home/proxyvnf/dashboard/config/db.php
    chown proxyvnf:proxyvnf /home/proxyvnf/dashboard/config/db.php
  SHELL

  # Install Composer and plugins
  config.vm.provision "shell", inline: <<-SHELL
    cd /tmp
    curl -s http://getcomposer.org/installer | php
    mv composer.phar /usr/local/bin/composer
    sudo -u proxyvnf composer global require "fxp/composer-asset-plugin:~1.1.0"
    cd /home/proxyvnf/dashboard
    sudo -u proxyvnf composer install
  SHELL

  # Migrate DB to MySQL
  # Create user admin:administrator
  # Populate the DB with the blacklisted domains (this step may take a long time)
  config.vm.provision "shell", inline: <<-SHELL
    cd /home/proxyvnf/dashboard
    sudo -u proxyvnf php yii migrate/up --interactive=0 --migrationPath=@vendor/dektrium/yii2-user/migrations
    sudo -u proxyvnf /home/proxyvnf/dashboard/yii createusers/create
    sudo -u proxyvnf /home/proxyvnf/dashboard/yii migrate --interactive=0
  SHELL

  # Check that all blacklists are loaded:
  # echo "DROP DATABASE dashboarddb; CREATE DATABASE dashboarddb;" | mysql -u root -p12345678
  # cd /home/proxyvnf/dashboard && sudo -u proxyvnf php yii migrate/up --interactive=0 --migrationPath=@vendor/dektrium/yii2-user/migrations && sudo -u proxyvnf /home/proxyvnf/dashboard/yii createusers/create && sudo -u proxyvnf /home/proxyvnf/dashboard/yii migrate --interactive=0
  # echo "SELECT COUNT(id) FROM dashboarddb.blacklist_domains;" | mysql -u root -p12345678
  # echo "`wc -l /etc/squidguard/blacklists/ads/domains` + `wc -l /etc/squidguard/blacklists/adult/domains`" | bc --mathlib

  # Apache rewrite module required for the dashboard
  config.vm.provision "shell", inline: <<-SHELL
    /usr/sbin/a2enmod rewrite
    systemctl restart apache2.service
  SHELL

  # Configure the apache document root
  config.vm.provision "file", source: "./conf/dashboard-dir.conf", destination: "dashboard-dir.conf"
  config.vm.provision "shell", inline: <<-'SHELL'
    mv /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/000-default.conf.dist
    cp /home/vagrant/dashboard-dir.conf /etc/apache2/conf-available/
    chown root:root /etc/apache2/conf-available/dashboard-dir.conf
    ln -sf /etc/apache2/conf-available/dashboard-dir.conf /etc/apache2/conf-enabled/dashboard-dir.conf
    ln -sf /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-enabled/000-default.conf
    sed -e 's/DocumentRoot[[:space:]]\+\/var\/www\/html$/DocumentRoot \/var\/www\/html\/dashboard\/web/g' /etc/apache2/sites-available/000-default.conf.dist > /etc/apache2/sites-available/000-default.conf
    ln -s /home/proxyvnf/dashboard /var/www/html/
    systemctl restart apache2.service
  SHELL

  # Reload the dashboard
  # ALWAYS RUN
  config.vm.provision "shell", run: "always", inline: <<-SHELL
    cd /home/proxyvnf/dashboard
    sudo -u proxyvnf git pull
  SHELL

  # TODO
  # Configure /etc/cloud
  # Start the cloud service?

end
