# ProXy as a Service (PXaaS) virtual Network Function (vNF)

Vagrant configuration file for PXaaS vNF provisioning in the framework of the [T-NOVA](http://t-nova.eu/) project.

A Debian Jessie box is built, which contains [Squid](http://www.squid-cache.org/), [SquidGuard](http://www.squidguard.org/) and a [dashboard](https://github.com/T-NOVA/Squid-dashboard).

The PXaaS vNF enables the provider to deploy a proxy/filtering/web access service on demand onto a cloud infrastructure.


## Host machine requirements

* KVM/VirtualBox
* [Vagrant](http://vagrantup.com)
* [vagrant-vbguest](https://github.com/dotless-de/vagrant-vbguest) plugin if provisioned with VirtualBox and [vagrant-libvirt](https://github.com/vagrant-libvirt/vagrant-libvirt) plugin if provisioned with libvirt

## Provision the VM

1. Clone the [proxy-build](https://github.com/T-NOVA/proxy-build) repository, and cd into the folder:

```sh
    git clone https://github.com/T-NOVA/proxy-build
    cd proxy-build
```

2. [Optional] To change the default IP, modify the [Vagrantfile](Vagrantfile):

```ruby
    config.vm.network "private_network", ip: "192.168.64.120"
```

3. Provision the VM:

```sh
    vagrant up --provider libvirt
```

Once provisioning is done, the VM should be up and running with [Squid](http://www.squid-cache.org/), [SquidGuard](http://www.squidguard.org/), preconfigured [blacklists](http://dsi.ut-capitole.fr/blacklists/index_en.php) and [dashboard](https://github.com/T-NOVA/Squid-dashboard).

4. [Optional] Change the `vagrant` user's password:

```sh
    vagrant ssh
    sudo passwd vagrant
```



# TODO

14. Allow apache2 to run sudo without providing password. This is useful when executing commands on Squid via the dashboard

```sh
    sudo touch /etc/sudoers.d/www-data
    sudo vim /etc/sudoers.d/www-data
```

and add:

```conf
    www-data ALL=(ALL) NOPASSWD:ALL
```

TODO
**ATTENTION** Major security issues with the above! Why not run apache2 as root anyway???????????


15. Test the application: http://192.168.64.120


## How to set up a development environment from an existing box

### Create a package from a box

```sh
    vagrant halt
    vagrant package
```

Creates `package.box`



## Deploy SquidGuard

Set up `/etc/squidguard/squidguard.conf`

```sh
    sudo cp -R ~/source/blacklists/* /var/lib/squidguard/db/
    sudo chown -R proxy:proxy /etc/squidguard
    sudo chown -R proxy:proxy /var/lib/squidguard
    sudo squidGuard -C all -d
```

5. Configure Squid. Add in `squid.conf`

6. Start squid

```sh
    sudo service start squid
```

## How to migrate a Virtualbox machine to OpenStack

```sh
    # [optional] Convert vmdk to qcow2
    qemu-img convert -f vmdk -O qcow2 image.vmdk image.qcow2
    # Unistall VirtualBox GuestAdditions
    sudo /opt/[VboxAddonsFolder]/uninstall.sh
```


