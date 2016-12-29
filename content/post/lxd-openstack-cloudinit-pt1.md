+++
date = "2016-12-15T11:40:59Z"
description = "Adventures with container networking"
title = "LXD, OpenStack and cloud-init - Part 1"
tags = [ "Cloud", "OpenStack", "LXD", "cloud-init" ]
series = [ "LXD" ]
+++

I've recently started playing with LXC containers, both on physical hardware and in the cloud. This blog is itself running in a LXC container running on a virtual machine in an OpenStack public cloud.

One of my main interests was to understand what the different patterns look like for external network access to services running containerised like this. By default LXD will configure an internal bridge, with DNSmasq providing internal IP addresses to containers. If you want to then open those services to external access, there are a number of different approaches you can take - you can use iptables to direct traffic from the host to the container, you can do L3 routing via your LXD host, or as I'm going to explore, you can bridge your containers onto your external network directly.  

If you're running on a LAN, then it's trivial to re-configure LXD to use a bridge with an external physical interface directly connected to the LAN. In that configuration, LXC containers will come up bridged directly to your physical network and can get an IP from an external DHCP server.

In my scenario, I'm not running on a LAN, my container host is going to be a virtual machine in a public OpenStack cloud. 

I started from the perspective of hoping I could put together a fully automatable workflow for handling this in a public cloud environment without native support for LXD ie. one without the nova-lxd service. If you're running nova-lxd in your OpenStack cloud then all this stuff 'just works', since the nova-lxd driver integrates directly with Neutron. If your public cloud doesn't have native support for LXD, then we need to get a bit more creative.

Ultimately what I'm trying to achieve here is a fully automatable workflow using only native OpenStack and LXD tooling to deliver LXD container services which can be consumed by the outside world. Everything that's required to do this should also be fully scriptable from a remote shell, giving us control over the automation.  

So taking my bridge pattern, the first thing I'm going to need to do is to bridge my virtual machines network interface, and switch LXD to use that bridge, so my containers are directly on the Neutron network.

 I had initially hoped I might be able to get an IP address from OpenStack's DHCP directly to my container, but once I'd thought that through properly I realised that wasn't going to work. Neutron uses fixed host definitions in DHCP, which are only created when a virtual machine is instantiated, so the Neutron provided DHCP server only serves addresses for virtual instances it knows about. 

My next thought was to create a network without OpenStack DHCP, and either run my own DHCP or use fixed addressing. This was also a dead end, since cloud init in cloud images generally expects to have the network interfaces definition delivered by the OpenStack metadata service, which again requires DHCP. It's possible to deliver this via an attached config drive, but your provider needs to support this, and mine doesn't currently. I could have created my own virtual machine images to support this model, but then this somewhat breaks the idea of what I'm trying to achieve here in terms of simple automation that will work anywhere. 

Finally, I decided to try and use a mixture of DHCP and fixed addressing within my OpenStack network, with the LXD virtual hosts getting their IP's from Neutron DHCP, and my containers being configured with static addresses, but bridged directly into the Neutron network. 

So here's how we go about doing that. 

My provider, like many OpenStack public cloud providers, automatically configures a default network, subnet and router within each of my projects. I'm not going to go into creating those here, and we'll assume you have a default network already created.

The first thing I want to do is to reduce the DHCP range for this subnet, so I've got some IP's which will definitely not get allocated via DHCP. 

Firstly, let's gather the ID's we'll need :

```
(openstack)MacBook-Pro:DCOS matt$ neutron net-list
+--------------------------------------+----------+-----------------------------------------------------+
| id                                   | name     | subnets                                             |
+--------------------------------------+----------+-----------------------------------------------------+
| 6751cb30-0aef-4d7e-94c3-ee2a09e705eb | external | 2af591ca-48ac-42b7-afc6-e691b3aa4c8a                |
|                                      |          | b839c2c8-94b9-4445-858d-1800b5fe3bbb                |
|                                      |          | e71eb7f6-400b-4d1f-a65b-9315ade67fe7                |
| be9b8434-7ec2-46e3-9c5f-5759440a2b06 | default  | a2b72e5b-48ec-46f0-ab40-df56e698bea2 192.168.0.0/24 |
+--------------------------------------+----------+-----------------------------------------------------+
```
Apologies for the formatting here, I'm still working this blog theme to display OpenStack CLI output correctly. The ID we're interested in is the bottom one for the default network. Now we've got that, let's get the subnet ID. 
```
(openstack)MacBook-Pro:DCOS matt$ neutron net-show be9b8434-7ec2-46e3-9c5f-5759440a2b06
+-----------------+--------------------------------------+
| Field           | Value                                |
+-----------------+--------------------------------------+
| admin_state_up  | True                                 |
| id              | be9b8434-7ec2-46e3-9c5f-5759440a2b06 |
| mtu             | 0                                    |
| name            | default                              |
| router:external | False                                |
| shared          | False                                |
| status          | ACTIVE                               |
| subnets         | a2b72e5b-48ec-46f0-ab40-df56e698bea2 |
| tenant_id       | c71e35d5f6034d45aad211e6a7784b6d     |
+-----------------+--------------------------------------+
```
If we take a look at that subnet, we can see the DHCP range is set to use the entire address range, which is the default behaviour.
```
(openstack)MacBook-Pro:DCOS matt$ neutron subnet-show a2b72e5b-48ec-46f0-ab40-df56e698bea2
+-------------------+--------------------------------------------------+
| Field             | Value                                            |
+-------------------+--------------------------------------------------+
| allocation_pools  | {"start": "192.168.0.2", "end": "192.168.0.254"} |
| cidr              | 192.168.0.0/24                                   |
| dns_nameservers   | 8.8.4.4                                          |
|                   | 8.8.8.8                                          |
| enable_dhcp       | True                                             |
| gateway_ip        | 192.168.0.1                                      |
| host_routes       |                                                  |
| id                | a2b72e5b-48ec-46f0-ab40-df56e698bea2             |
| ip_version        | 4                                                |
| ipv6_address_mode |                                                  |
| ipv6_ra_mode      |                                                  |
| name              | default                                          |
| network_id        | be9b8434-7ec2-46e3-9c5f-5759440a2b06             |
| subnetpool_id     |                                                  |
| tenant_id         | c71e35d5f6034d45aad211e6a7784b6d                 |
+-------------------+--------------------------------------------------+
```
Let's reduce the size of that allocation pool. All I will have in that pool are any LXD hosts I bring up, so it doesn't need to be very big
```
(openstack)MacBook-Pro:DCOS matt$ neutron subnet-update --allocation-pool start=192.168.0.2,end=192.168.0.20 a2b72e5b-48ec-46f0-ab40-df56e698bea2
Updated subnet: a2b72e5b-48ec-46f0-ab40-df56e698bea2
```
Now when we look at that subnet again, we can see the allocation pool has been reduced to where we want it
```
(openstack)MacBook-Pro:DCOS matt$ neutron subnet-show a2b72e5b-48ec-46f0-ab40-df56e698bea2
+-------------------+-------------------------------------------------+
| Field             | Value                                           |
+-------------------+-------------------------------------------------+
| allocation_pools  | {"start": "192.168.0.2", "end": "192.168.0.20"} |
| cidr              | 192.168.0.0/24                                  |
| dns_nameservers   | 8.8.4.4                                         |
|                   | 8.8.8.8                                         |
| enable_dhcp       | True                                            |
| gateway_ip        | 192.168.0.1                                     |
| host_routes       |                                                 |
| id                | a2b72e5b-48ec-46f0-ab40-df56e698bea2            |
| ip_version        | 4                                               |
| ipv6_address_mode |                                                 |
| ipv6_ra_mode      |                                                 |
| name              | default                                         |
| network_id        | be9b8434-7ec2-46e3-9c5f-5759440a2b06            |
| subnetpool_id     |                                                 |
| tenant_id         | c71e35d5f6034d45aad211e6a7784b6d                |
+-------------------+-------------------------------------------------+
```
The next thing we need to do is spin up our first LXD host in this subnet. 

For our host virtual machine to work as we want in our container infrastructure, we need to change the default networking, from a single virtual ethernet interface to a bridge containing that virtual ethernet interface, with the DHCP address on the bridge rather than the interface directly. We can then configure LXD to use our bridge, and containers will be directly on the Neutron network. OpenStack doesn't support creating instances with bridged networking, so we'll have to automate this in a different way. 

In order to do this, we'll leverage some of the magic of cloud-init, passing configuration in when we boot our instance. For those of you not too familiar with cloud-init, it's the mechanism by which networking generally gets configured in cloud platforms, but it can do a ton of other cool stuff. It's well worth a read of the [docs](https://cloudinit.readthedocs.io/en/latest).

To do interesting stuff with cloud-init, we need to pass in some yaml to nova when we boot our instance, and here's what I'm passing to it, in a local file I've imaginatively named config.yaml :

```
#cloud-config
## vim: syntax=yaml
##
packages:
 - bridge-utils
 - dnsmasq
write_files:
 - path: /etc/network/interfaces
   content: |
    # Re-configure for LXD bridge
    auto ens3
    auto br0
    iface br0 inet dhcp
        bridge_ports ens3

    iface ens3 inet manual
        up ifconfig ens3 promisc up
 - path: /etc/systemd/network/10-bridge.link
   content: |
    [Match]
    OriginalName=br0

    [Link]
    Name=br0
 - path: /etc/dnsmasq.d/lxd.conf
   content: |
    dhcp-range=192.168.0.0,static,255.255.255.0
    dhcp-option=3,192.168.0.1
    no-hosts
lxd:
   init:
        storage_backend: dir
   bridge:
        mode: existing
        name: br0
        ipv4_nat: false
power_state:
   delay: "now"
   mode: reboot
   message: Bye Bye
   timeout: 60
   condition: True
```

Let's look at what this will actually do - firstly we have the section: 

```
#cloud-config
```

That's required syntax to get the yaml interpreted by cloud-init. 

Because we're going to be using bridges, I want the user-space tools for viewing and manipulating bridges, so we install the bridge-utils package

```
packages:
 - bridge-utils
 - dnsmasq
```

This is the standard pattern for a cloud-init entry, the top level key is the cloud-init module to use, and the data following is the configuration for that module. In this case, the packages module, which takes an array of package names as its data.

We then want to write a couple of files to the filesystem, using the aptly named write_files module.

```
write_files:
 - path: /etc/network/interfaces
   content: |
    # Re-configure for LXD bridge
    auto ens3
    auto br0
    iface br0 inet dhcp
        bridge_ports ens3

    iface ens3 inet manual
        up ifconfig ens3 promisc up
 - path: /etc/systemd/network/10-bridge.link
   content: |
    [Match]
    OriginalName=br0

    [Link]
    Name=br0
 - path: /etc/dnsmasq.d/lxd.conf
   content: |
    dhcp-range=192.168.0.0,static,255.255.255.0
    dhcp-option=3,192.168.0.1
    no-hosts
```

The first one is our new network interfaces file with our bridge configuration, all fairly standard for ethernet bridging, with the exception being that because the ens3 interface is actually virtual, it needs to be in promiscous mode. 

The second file is to solve some weirdness I had on Ubuntu 16.04 with systemd-udevd and the kernel getting confused with [renaming](https://www.freedesktop.org/wiki/Software/systemd/PredictableNetworkInterfaceNames/) of the bridge interface. In order to workaround this, we [override](https://www.freedesktop.org/software/systemd/man/systemd.link.html) systemd and tell it not to change the name of the bridge. 

The final file is additional configuration for a local DNSmasq instance to provide static IP's to my containers. I'll talk more about this in my [second](/post/lxd-openstack-cloudinit-pt2) blog post, but this is one strategy that you can use to assign static IP's in this particular scenario. 

We also want to configure LXD automatically, rather than having to manually do :

```
lxd init
```

so we use the LXD cloud-init module, telling LXD to use files as its storage backend, use our existing bridge with no DHCP and no NAT.

```
lxd:
   init:
        storage_backend: dir
   bridge:
        mode: existing
        name: br0
        ipv4_nat: false
```

Finally, in order to get our new bridge networking in place, let's reboot the machine once it's finished, using the power_state module

```
power_state:
   delay: "now"
   mode: reboot
   message: Bye Bye
   timeout: 60
   condition: True
```
This basically tells cloud-init to wait 60 seconds for cloud-init to finish, and then run 
```
shutdown -r now "Bye Bye"
```

The nice thing about all of these commands is that cloud-init is state aware, so this stuff will only happen once.

Now we've got all our cloud-init configuration in place, we pass that yaml file as an argument to our nova boot command. 

```
(openstack)MacBook-Pro:DCOS matt$ nova boot --image ef24441d-01ad-4006-b63d-6da67b7f1348 --flavor 196235bc-7ca5-4085-ac81-7e0242bda3f9 --user-data ~/LXD/DC/config.yaml --key-name datacentred lxd0
+--------------------------------------+------------------------------------------------------------------+
| Property                             | Value                                                            |
+--------------------------------------+------------------------------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                                           |
| OS-EXT-AZ:availability_zone          | nova                                                             |
| OS-EXT-STS:power_state               | 0                                                                |
| OS-EXT-STS:task_state                | scheduling                                                       |
| OS-EXT-STS:vm_state                  | building                                                         |
| OS-SRV-USG:launched_at               | -                                                                |
| OS-SRV-USG:terminated_at             | -                                                                |
| accessIPv4                           |                                                                  |
| accessIPv6                           |                                                                  |
| adminPass                            | VzUTXaAu5haG                                                     |
| config_drive                         |                                                                  |
| created                              | 2016-12-23T11:54:36Z                                             |
| flavor                               | dc1.2x4.40 (196235bc-7ca5-4085-ac81-7e0242bda3f9)                |
| hostId                               |                                                                  |
| id                                   | f3ecfa36-be6a-4432-bacc-6bc9a7d0768e                             |
| image                                | Ubuntu 16.04 LTS (Xenial) (ef24441d-01ad-4006-b63d-6da67b7f1348) |
| key_name                             | datacentred                                                      |
| metadata                             | {}                                                               |
| name                                 | lxd0                                                             |
| os-extended-volumes:volumes_attached | []                                                               |
| progress                             | 0                                                                |
| security_groups                      | default                                                          |
| status                               | BUILD                                                            |
| tenant_id                            | c71e35d5f6034d45aad211e6a7784b6d                                 |
| updated                              | 2016-12-23T11:54:36Z                                             |
| user_id                              | 73e25cbdf18a4785a1341fe59a06a9da                                 |
+--------------------------------------+------------------------------------------------------------------+
```

Once the instance has booted, cloud-init will put in place all of our required configuration, reboot the instance, and we'll have a working LXD host with bridged networking ready to run containers. In the [next](/post/lxd-openstack-cloudinit-pt2) exciting installment, I'll move on to running containers, configuring Neutron to allow them access to the network, and using cloud-init again at the container level to configure the container on launch. 

