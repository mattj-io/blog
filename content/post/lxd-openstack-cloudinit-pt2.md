+++
title = "LXD, OpenStack and cloud-init - Part 2"
description = "More adventures with container networking"
date = "2016-12-18T13:03:07Z"

+++

So in my previous post, we automatically deployed and configured an LXD host into our OpenStack public cloud tenancy, using only the native features of OpenStack and cloud-init. 

Let's take a look at that instance and confirm everything is as it should be.

Firstly we'll need to assign some additional security rules to allow external access. By default most OpenStack clouds don't allow any access, to give the user the most flexibility in defining their security goals. YMMV here depending on what your provider sets up by default. 

In my case, we'll create a new security group to handle SSH access. There are tons of different ways you can manage this, security groups per role, adding rules to the default, but here I'll keep my services in separate security groups. 

```
(openstack)MacBook-Pro:DCOS matt$ nova secgroup-create ssh_access SSH
+--------------------------------------+------------+-------------+
| Id                                   | Name       | Description |
+--------------------------------------+------------+-------------+
| 980deb5f-877b-460e-b11b-f0bfc6140931 | ssh_access | SSH         |
+--------------------------------------+------------+-------------+
```

Now we need to add a rule to the security group for SSH access. I'm going to allow from any address, but in production situations you may want to restrict that a bit more.

```
(openstack)MacBook-Pro:DCOS matt$ nova secgroup-add-rule ssh_access tcp 22 22 0.0.0.0/0
+-------------+-----------+---------+-----------+--------------+
| IP Protocol | From Port | To Port | IP Range  | Source Group |
+-------------+-----------+---------+-----------+--------------+
| tcp         | 22        | 22      | 0.0.0.0/0 |              |
+-------------+-----------+---------+-----------+--------------+
```

And now we'll add that to our instance

```
(openstack)MacBook-Pro:DCOS matt$ nova add-secgroup 1275cf9b-2590-4423-b720-3ea86354f5d2 ssh_access
```

and confirm it's been assigned

```
(openstack)MacBook-Pro:DCOS matt$ nova show 1275cf9b-2590-4423-b720-3ea86354f5d2
+--------------------------------------+------------------------------------------------------------------+
| Property                             | Value                                                            |
+--------------------------------------+------------------------------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                                           |
| OS-EXT-AZ:availability_zone          | Production                                                       |
| OS-EXT-STS:power_state               | 1                                                                |
| OS-EXT-STS:task_state                | -                                                                |
| OS-EXT-STS:vm_state                  | active                                                           |
| OS-SRV-USG:launched_at               | 2016-12-17T15:58:50.000000                                       |
| OS-SRV-USG:terminated_at             | -                                                                |
| accessIPv4                           |                                                                  |
| accessIPv6                           |                                                                  |
| config_drive                         |                                                                  |
| created                              | 2016-12-17T15:58:37Z                                             |
| default network                      | 192.168.0.11                                                     |
| flavor                               | dc1.1x1.20 (8e6069a3-d8c6-4741-8e0d-6373b2ca38cc)                |
| hostId                               | a820834ac02c3e42d69ee91115afb99a691730b9d6ef73f89d4bb73b         |
| id                                   | 1275cf9b-2590-4423-b720-3ea86354f5d2                             |
| image                                | Ubuntu 16.04 LTS (Xenial) (ef24441d-01ad-4006-b63d-6da67b7f1348) |
| key_name                             | datacentred                                                      |
| metadata                             | {}                                                               |
| name                                 | lxd_test                                                         |
| os-extended-volumes:volumes_attached | []                                                               |
| progress                             | 0                                                                |
| security_groups                      | default, ssh_access                                              |
| status                               | ACTIVE                                                           |
| tenant_id                            | c71e35d5f6034d45aad211e6a7784b6d                                 |
| updated                              | 2016-12-17T15:58:50Z                                             |
| user_id                              | 73e25cbdf18a4785a1341fe59a06a9da                                 |
+--------------------------------------+------------------------------------------------------------------+
```

Next we need to associate a floating IP so we can access the instance externally. First let's check our project allocation to find an unused one

```
(openstack)MacBook-Pro:DCOS matt$ nova floating-ip-list
+--------------------------------------+----------------+--------------------------------------+-------------+----------+
| Id                                   | IP             | Server Id                            | Fixed IP    | Pool     |
+--------------------------------------+----------------+--------------------------------------+-------------+----------+
| 7c202d8a-f924-4bc1-96d9-ec9de55145dd | 185.98.150.244 | -                                    | -           | external |
| 8361ba16-8088-47f5-bf52-e7929a0dd4f3 | 185.98.151.85  | -                                    | -           | external |
+--------------------------------------+----------------+--------------------------------------+-------------+----------+
```
And we'll add one of those to our instance

```
(openstack)MacBook-Pro:DCOS matt$ nova floating-ip-associate 1275cf9b-2590-4423-b720-3ea86354f5d2 185.98.151.85
```

Now we should be able to access our instance, so let's check the expected configuration for our network and LXD that we created using cloud-init is in place. Remember we need to use the keypair which we configured the instance with when we created it. 

```
MacBook-Pro:~ matt$ ssh -i ~/.ssh/mattj_dc ubuntu@185.98.151.85 'ifconfig'
br0       Link encap:Ethernet  HWaddr fa:16:3e:38:88:a9  
          inet addr:192.168.0.11  Bcast:192.168.0.255  Mask:255.255.255.0
          inet6 addr: fe80::f816:3eff:fe38:88a9/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:5189 errors:0 dropped:0 overruns:0 frame:0
          TX packets:2419 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:64974301 (64.9 MB)  TX bytes:227013 (227.0 KB)
          
ens3      Link encap:Ethernet  HWaddr fa:16:3e:38:88:a9  
          inet6 addr: fe80::f816:3eff:fe38:88a9/64 Scope:Link
          UP BROADCAST RUNNING PROMISC MULTICAST  MTU:1500  Metric:1
          RX packets:5200 errors:0 dropped:0 overruns:0 frame:0
          TX packets:2426 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:65049069 (65.0 MB)  TX bytes:227523 (227.5 KB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:180 errors:0 dropped:0 overruns:0 frame:0
          TX packets:180 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:13360 (13.3 KB)  TX bytes:13360 (13.3 KB)
MacBook-Pro:~ matt$ ssh -i ~/.ssh/mattj_dc ubuntu@185.98.151.85 'lxc profile show default'
name: default
config: {}
description: Default LXD profile
devices:
  eth0:
    name: eth0
    nictype: bridged
    parent: br0
    type: nic
```

All good so far, our networking configuration is correct, and LXD is configured as we'd expect.

Let's move on to creating the configuration for our containers. 

Now we could build our own container images containing everything we need, but my containers will be fairly simple, serving a single service. For this, we will use a standard Ubuntu container image, and just configure the container at launch time using cloud-init. 

I'm going to use additional per-container profiles here, but you could just initialise the container and then modify the local config.

First let's create a new profile for our container

```
MacBook-Pro:~ matt$ ssh -i ~/.ssh/mattj_dc ubuntu@185.98.151.85 'lxc profile create hugo'
Profile hugo created
```

We're calling it hugo since this container will run the [hugo](https://gohugo.io/) static website engine.

There's some [discussion](https://github.com/lxc/lxd/issues/1259) around the idiomatics of creating LXD containers with static IP's - the recommended way appears to be to configure the network interfaces file inside the container and then restart it, but to me this seems to add an additional step of complexity where it's not needed.

It feels like this should be possible by injecting the interfaces configuration in the user.meta-data field for cloud-init, but I couldn't get that to work with a standard Ubuntu 16.04 LXD image. 

Another option is to use the unsupported raw.lxc configuration flags, which inject the networking configuration directly into the kernel. To set the configuration for this over ssh, we need to pipe it into the lxc config command as it's multi-line.

```
MacBook-Pro:~ matt$ ssh -i ~/.ssh/mattj_dc ubuntu@185.98.151.85 '(
cat << EOF
lxc.network.0.ipv4 = 192.168.0.100/24
lxc.network.0.ipv4.gateway = 192.168.0.1
EOF                                     
) | lxc profile set hugo raw.lxc -' 
```

If we were to use the raw lxc approach, we would also need to make one change to the default profile. Since with this approach we would be setting static IP addresses on container boot, we would want to make sure cloud-init doesn't then try to configure the interface via DHCP, which can cause cloud-init to take a long time to execute.

```
MacBook-Pro:~ matt$ ssh -i ~/.ssh/mattj_dc ubuntu@185.98.151.85 'lxc profile set default user.network_mode link-local'
``` 

In this mode you'd also need to set the DNS for the container very early in the boot process, because a lot of cloud-init depends on this and will fail without it ( Hint - the cloud-init bootcmd module is your friend )

The raw lxc approach works well, at the small cost of a few more lines of cloud-init, but since all of that is unsupported, we never know if this facility might be removed without warning in later versions, so we'll take an alternative approach. 

When creating it's own bridge, LXD can spawn a DNSmasq instance onto the bridge and provide DHCP to containers. However, when we use our own bridge, the assumption is that we are providing DHCP externally and LXD won't configure this. 

Now remember our br0 is bridged at Layer 2 directly onto the Neutron provided network, which has it's own DHCP server. We definitely don't want to be sending DHCP from inside our instance, as this could get potentially very confusing for other virtual machines. 

However, DNSmasq can be configured so that it will only respond to specific requests, which as long as we're careful shouldn't interfere with the Neutron DHCP server. 

As I explained in the [first](/post/lxd-openstack-cloudinit-pt1) part of this blog, during the cloud-init phase we passed some DNSmasq configuration into our virtual machine, so let's take a look at that again:

```
 - path: /etc/dnsmasq.d/lxd.conf
   content: |
    dhcp-range=192.168.0.0,static,255.255.255.0
    dhcp-option=3,192.168.0.1
    no-hosts
```

What this piece of configuration does is tell DNSmasq not to hand out any dynamic addresses,and to set the default gateway for any DHCP responses to 192.168.0.1, otherwise it will use the host it's running on as the default. DNSmasq can also be configured to use /etc/hosts as it's configuration for which hosts to assign addresses to, so the no-hosts setting tells it to ignore this. At this point, DNSmasq will not respond to any DHCP requests. 

When we want to add a host that DNSmasq will respond to, we just add a configuration line to our file /etc/dnsmasq.d/lxd.conf :

```
dhcp-host=hugo,192.168.0.100
```

Now when DNSmasq sees a DHCP request with hostname set to hugo, it will respond with an address of 192.168.0.100. We don't need to worry about the Neutron DHCP server responding, as it won't respond to any requests it doesn't already know about, and as long as we don't use the same hostnames for containers and virtual machines, we should be OK.  

The nice thing about this is that we also get DNS for free, since DNSmasq will serve DNS too, using the local DNS settings on our LXD host as it's upstream resolver. 
 
Now our container will come up with the correct IP address assigned, and we can add the rest of our configuration. You can edit the profile interactively using :

```
lxc profile edit hugo
```

but since I'm aiming for a fully automatable workflow, I'm just going to run commands over SSH.  

The cloud-init configuration is all part of a single key in the profile, called user.user-data, so we need to add it all at the same time. We could also define all of this in an XML file, copy it to the machine and pass it in as input to our command.

```
MacBook-Pro:~ matt$ ssh -i ~/.ssh/mattj_dc ubuntu@185.98.151.85 '(
cat << EOF
#cloud-config
packages:
  - hugo
  - git
runcmd:
  - git clone https://github.com/mattj-io/blog.git /home/ubuntu/blog
EOF
) | lxc profile set hugo user.user-data -'
```

Basically what we're doing here is install the packages we need, using the packages module. We then pull down the website we want to serve from github.

Now, if we start the container we can see it has come up with the correct IP address we want :

```
MacBook-Pro:~ matt$ ssh -i ~/.ssh/mattj_dc ubuntu@185.98.151.85 'lxc start hugo'
MacBook-Pro:~ matt$ ssh -i ~/.ssh/mattj_dc ubuntu@185.98.151.85 'lxc list hugo'
+------+---------+----------------------+------+------------+-----------+
| NAME |  STATE  |         IPV4         | IPV6 |    TYPE    | SNAPSHOTS |
+------+---------+----------------------+------+------------+-----------+
| hugo | RUNNING | 192.168.0.100 (eth0) |      | PERSISTENT | 0         |
+------+---------+----------------------+------+------------+-----------+
```

However, at this point, whilst we can ping the container from our LXD host, the container itself still hasn't got outbound connectivity. The reason for this is that Neutron by default is configured for port level security. This takes the form of IPtables rules on the virtual router port of the instance, which only allow traffic with the MAC/IP combination of the virtual machine to pass through. 

If your provider has the [port security extension](https://wiki.openstack.org/wiki/Neutron/ML2PortSecurityExtensionDriver) enabled, then you can turn off the port security features either at the port level or at the network level. My provider doesn't currently have this enabled, so I need a slightly more involved workaround. 

In the default configuration, Neutron ports have the concept of allowed address pairs, which means we can add up to 10 entries which are allowed access across the port. IP addresses can be wildcarded in this configuration, but unfortunately not MAC addresses, so until my provider enables port security extension, I'm restricted to 10 containers per host. 

Here's how we allow the container access to the port. 

First let's find the port we need, we're looking for our virtual machine IP address of 192.168.0.11 :

```
(openstack)MacBook-Pro:DCOS matt$ neutron port-list
+--------------------------------------+------+-------------------+-------------------------------------------------------------------------------------+
| id                                   | name | mac_address       | fixed_ips                                                                           |
+--------------------------------------+------+-------------------+-------------------------------------------------------------------------------------+
| 6b63c522-2266-46ca-b4ae-830344fe8948 |      | fa:16:3e:38:88:a9 | {"subnet_id": "a2b72e5b-48ec-46f0-ab40-df56e698bea2", "ip_address": "192.168.0.11"} |
| 9fae0b86-9f29-4709-a170-81ceb5a4a4d7 |      | fa:16:3e:11:bd:6d | {"subnet_id": "a2b72e5b-48ec-46f0-ab40-df56e698bea2", "ip_address": "192.168.0.1"}  |
| d2796309-92d0-418e-96c5-d255a72a2ec5 |      | fa:16:3e:4b:bd:86 | {"subnet_id": "a2b72e5b-48ec-46f0-ab40-df56e698bea2", "ip_address": "192.168.0.2"}  |
| d8de4921-785a-45b6-b25d-c88fc976176d |      | fa:16:3e:74:70:4b | {"subnet_id": "a2b72e5b-48ec-46f0-ab40-df56e698bea2", "ip_address": "192.168.0.3"}  |
| e6722779-ccfc-4377-93a0-18d47f350848 |      | fa:16:3e:bd:66:a7 | {"subnet_id": "a2b72e5b-48ec-46f0-ab40-df56e698bea2", "ip_address": "192.168.0.4"}  |
+--------------------------------------+------+-------------------+-------------------------------------------------------------------------------------+
```

Now let's take a look at that port in more detail

```
(openstack)MacBook-Pro:DCOS matt$ neutron port-show 6b63c522-2266-46ca-b4ae-830344fe8948
+-----------------------+-------------------------------------------------------------------------------------+
| Field                 | Value                                                                               |
+-----------------------+-------------------------------------------------------------------------------------+
| admin_state_up        | True                                                                                |
| allowed_address_pairs |                                                                                     |
| binding:vnic_type     | normal                                                                              |
| device_id             | 1275cf9b-2590-4423-b720-3ea86354f5d2                                                |
| device_owner          | compute:None                                                                        |
| extra_dhcp_opts       |                                                                                     |
| fixed_ips             | {"subnet_id": "a2b72e5b-48ec-46f0-ab40-df56e698bea2", "ip_address": "192.168.0.11"} |
| id                    | 6b63c522-2266-46ca-b4ae-830344fe8948                                                |
| mac_address           | fa:16:3e:38:88:a9                                                                   |
| name                  |                                                                                     |
| network_id            | be9b8434-7ec2-46e3-9c5f-5759440a2b06                                                |
| security_groups       | 980deb5f-877b-460e-b11b-f0bfc6140931                                                |
|                       | e70972a7-0217-48f3-939c-7fdd3e914c75                                                |
| status                | ACTIVE                                                                              |
| tenant_id             | c71e35d5f6034d45aad211e6a7784b6d                                                    |
+-----------------------+-------------------------------------------------------------------------------------+
```
As we can see the allowed address pairs value is currently empty, so let's update it for our container. We'll let the IP be anything in our range. 

```
(openstack)MacBook-Pro:DCOS matt$ neutron port-update 6b63c522-2266-46ca-b4ae-830344fe8948 --allowed-address-pairs type=dict list=true mac_address=00:16:3e:84:b0:10,ip_address=192.168.0.0/24
Updated port: 6b63c522-2266-46ca-b4ae-830344fe8948
```

Now when we look at that port again, we'll see that it's been updated to allow that combination of MAC and IP to use the port

```
(openstack)MacBook-Pro:DCOS matt$ neutron port-show 6b63c522-2266-46ca-b4ae-830344fe8948
+-----------------------+-------------------------------------------------------------------------------------+
| Field                 | Value                                                                               |
+-----------------------+-------------------------------------------------------------------------------------+
| admin_state_up        | True                                                                                |
| allowed_address_pairs | {"ip_address": "192.168.0.0/24", "mac_address": "00:16:3e:84:b0:10"}                |
| binding:vnic_type     | normal                                                                              |
| device_id             | 1275cf9b-2590-4423-b720-3ea86354f5d2                                                |
| device_owner          | compute:None                                                                        |
| extra_dhcp_opts       |                                                                                     |
| fixed_ips             | {"subnet_id": "a2b72e5b-48ec-46f0-ab40-df56e698bea2", "ip_address": "192.168.0.11"} |
| id                    | 6b63c522-2266-46ca-b4ae-830344fe8948                                                |
| mac_address           | fa:16:3e:38:88:a9                                                                   |
| name                  |                                                                                     |
| network_id            | be9b8434-7ec2-46e3-9c5f-5759440a2b06                                                |
| security_groups       | 980deb5f-877b-460e-b11b-f0bfc6140931                                                |
|                       | e70972a7-0217-48f3-939c-7fdd3e914c75                                                |
| status                | ACTIVE                                                                              |
| tenant_id             | c71e35d5f6034d45aad211e6a7784b6d                                                    |
+-----------------------+-------------------------------------------------------------------------------------+
```

And now if we test from our container host, our container has network access !

```
ubuntu@lxd-test:~$ lxc exec hugo bash
root@hugo:~# ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=57 time=7.33 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=57 time=6.83 ms
^C
--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 6.837/7.086/7.335/0.249 ms
```

Bear in mind this is very simplistic and only works well for self-contained services. What we really need is for every service to suport gossip type protocols, and can announce capabilities and monitoring points. 
