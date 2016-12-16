+++
date = "2016-12-15T11:40:59Z"
description = "First steps with LXD"
title = "Adventures with container networking"

+++

I've recently started playing with LXC containers, both on physical hardware and in the cloud. This blog is itself running in a LXC container running on a virtual machine in an OpenStack public cloud.

One of my main interests was to understand what the different patterns look like for external network access to services running containerised like this. By default LXD will configure an internal bridge, with DNSmasq providing internal IP addresses to containers. If you want to then open those services to external access, there are a number of different approaches you can take. 

I started from the perspective of hoping I could put together a fully automated workflow for handling this in a public cloud environment, ie. one without the nova-lxd service. If you're running nova-lxd in your OpenStack cloud then all this stuff 'just works', since the nova-lxd driver integrates directly with neutron. If your public cloud doesn't have this service, then your options are slightly more limited. 

If you're running on a LAN, then it's trivial to re-configure LXD to use a bridge with an external physical interface directly connected to the LAN. In this configuration, LXC containers will come up bridged directly to your physical network and can get an IP from an external DHCP server.

So taking that pattern, my first exploration was to bridge my virtual machine interface, and switch LXD to use that bridge. Before I'd thought that through properly, I had hoped I might be able to get an address from DHCP directly to my container, but Neutron uses fixed host definitions in DHCP, which are only created when a virtual machine is instantiated, so that won't actually work. My next thought was to create a network without OpenStack DHCP, and either run my own DHCP or use fixed addressing. This was also a slight brick wall, since cloud init in cloud images generally expects to have the network interfaces definition delivered by the OpenStack metadata service, which again requires DHCP. It's possible to deliver this via an attached config drive, but your provider needs to support this, and mine doesn't currently. I could have created my own images, but then this somewhat breaks the idea of what I'm trying to achieve here in terms of simple automation   

Still, it's an interesting thought process to work through how we achieve the first part of this :

Firstly, let's bring up an instance in our OpenStack cloud, I'm going to assume we already have a network created. 

```
nova boot --image ef24441d-01ad-4006-b63d-6da67b7f1348 --flavor 8e6069a3-d8c6-4741-8e0d-6373b2ca38cc --key-name yourkeyhere lxd_test
```

In my case this is a 2 core 4 GB RAM instance, running Ubuntu Xenial. I'm also injecting a pre-created keypair so I can ssh into the box. 
