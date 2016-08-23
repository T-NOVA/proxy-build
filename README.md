# ProXy as a Service (PXaaS) virtual Network Function (vNF)

Vagrant configuration file for PXaaS vNF provisioning in the framework of the [T-NOVA](http://t-nova.eu/) project.

A Debian Jessie box is built, which contains [Squid](http://www.squid-cache.org/), [SquidGuard](http://www.squidguard.org/) and a [dashboard](https://github.com/T-NOVA/Squid-dashboard) to control rules, blacklists and ACLs.

The PXaaS vNF enables the provider to deploy a web proxy/filtering/anonymity service on demand onto a cloud platform.

Squid typically uses only a single processor, even on a multi-processor machine. To get increased web-caching performance, it is better to scale the web cache out across multiple vNFs.


## Host machine requirements

* KVM/VirtualBox
* [Vagrant](http://vagrantup.com)
* [vagrant-vbguest](https://github.com/dotless-de/vagrant-vbguest) plugin if provisioned with VirtualBox or [vagrant-libvirt](https://github.com/vagrant-libvirt/vagrant-libvirt) plugin if provisioned with libvirt
* [vagrant-hosts](https://github.com/oscar-stack/vagrant-hosts) plugin for managing local DNS resolution


## Provision the VM

The VM is built using the [official OpenStack guide](http://docs.openstack.org/image-guide/openstack-images.html).

**Step 1.** Clone the [proxy-build](https://github.com/T-NOVA/proxy-build) repository, and cd into the folder:

```sh
    git clone https://github.com/T-NOVA/proxy-build && cd proxy-build
```

**Step 2.** Download the vagrant box

Debian (login `debian`):

```sh
    vagrant box add debian-openstack http://cdimage.debian.org/cdimage/openstack/current/debian-8.5.0-openstack-amd64.qcow2
```

Alpine :

```sh
    vagrant box add maier/alpine-3.4-x86_64
```

Ubuntu (login `ubuntu`):

```sh
    vagrant box add xenial-daily https://cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-amd64-vagrant.box
```

or `https://cloud-images.ubuntu.com/daily/server/xenial/current/xenial-server-cloudimg-amd64-disk1.img`

**Step 3.** Provision the VM:

```sh
    vagrant up --provider libvirt
```

Once provisioning is done, the VM should be up and running with [Squid](http://www.squid-cache.org/), [SquidGuard](http://www.squidguard.org/), preconfigured [blacklists](http://dsi.ut-capitole.fr/blacklists/index_en.php) and [dashboard](https://github.com/T-NOVA/Squid-dashboard).

**Step 4.** Change the `vagrant` user's password so users can log in:

```sh
    vagrant ssh
    sudo passwd vagrant
```

or, distribute the generated ssh key.

**Step 5.** On your dev machine, access the dashboard at http://pxaas. To test the Squid proxy do:

```sh
    curl -x pxaas:8000 http://google.com
```

To test that site blocking is enabled do:

```sh
    curl -x pxaas:8000 http://facebook.com
```


## Prepare VM for OpenStack deployment

Reset the virtual machine details to prepare for OpenStack deployment:

```sh
    virt-sysprep -d pxaas
```


## Reload the VM configuration

The VM is configured to pull new commits into the dashboard directory on reload. Execute:

```sh
    vagrant reload
```


## Local VM development

To work on the local VM, first provision it and then use a sandbox environment to test, commit or rollback additional configuration.

**Step 1.** Install the sahara and fog plugins

```sh
    vagrant plugin install sahara
    vagrant plugin install fog
```

**Step 2.** Enter sandbox mode

```sh
    vagrant sandbox on
```

**Step 3.** Save the changes permanently

```sh
    vagrant sandbox commit
```

OR **Step 3.** Rollback the changes

```sh
    vagrant sandbox rollback
```

**Step 4.** Exit sandbox mode

```sh
    vagrant sandbox off
```

