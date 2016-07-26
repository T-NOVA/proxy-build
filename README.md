# ProXy as a Service (PXaaS) virtual Network Function (vNF)

Vagrant configuration file for PXaaS vNF provisioning in the framework of the [T-NOVA](http://t-nova.eu/) project.

A Debian Jessie box is built, which contains [Squid](http://www.squid-cache.org/), [SquidGuard](http://www.squidguard.org/) and a [dashboard](https://github.com/T-NOVA/Squid-dashboard).

The PXaaS vNF enables the provider to deploy a proxy/filtering/web access service on demand onto a cloud infrastructure.


## Host machine requirements

* KVM/VirtualBox
* [Vagrant](http://vagrantup.com)
* [vagrant-vbguest](https://github.com/dotless-de/vagrant-vbguest) plugin if provisioned with VirtualBox and [vagrant-libvirt](https://github.com/vagrant-libvirt/vagrant-libvirt) plugin if provisioned with libvirt

## Provision the VM

**Step 1.** Clone the [proxy-build](https://github.com/T-NOVA/proxy-build) repository, and cd into the folder:

```sh
    git clone https://github.com/T-NOVA/proxy-build && cd proxy-build
```

_Optional_ To change the default IP, modify the [Vagrantfile](Vagrantfile):

```ruby
    config.vm.network "private_network", ip: "192.168.64.120"
```

**Step 2.** Provision the VM:

```sh
    vagrant up --provider libvirt
```

Once provisioning is done, the VM should be up and running with [Squid](http://www.squid-cache.org/), [SquidGuard](http://www.squidguard.org/), preconfigured [blacklists](http://dsi.ut-capitole.fr/blacklists/index_en.php) and [dashboard](https://github.com/T-NOVA/Squid-dashboard).

**Step 3.** Change the `vagrant` user's password so users can log in:

```sh
    vagrant ssh
    sudo passwd vagrant
```

**Step 4.** Access the application at http://192.168.64.120

