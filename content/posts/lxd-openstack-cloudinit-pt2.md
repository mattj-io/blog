+++
title = "LXD, OpenStack and cloud-init - Part 2"
description = "More adventures with container networking"
date = "2016-12-18T13:03:07Z"
tags = [ "Cloud", "OpenStack", "LXD", "cloud-init" ]
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
(openstack)MacBook-Pro:DCOS matt$ nova add-secgroup f3ecfa36-be6a-4432-bacc-6bc9a7d0768e ssh_access
```

and confirm it's been assigned

```
(openstack)MacBook-Pro:DCOS matt$ nova show f3ecfa36-be6a-4432-bacc-6bc9a7d0768e
+--------------------------------------+------------------------------------------------------------------+
| Property                             | Value                                                            |
+--------------------------------------+------------------------------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                                           |
| OS-EXT-AZ:availability_zone          | Production                                                       |
| OS-EXT-STS:power_state               | 1                                                                |
| OS-EXT-STS:task_state                | -                                                                |
| OS-EXT-STS:vm_state                  | active                                                           |
| OS-SRV-USG:launched_at               | 2016-12-23T11:54:45.000000                                       |
| OS-SRV-USG:terminated_at             | -                                                                |
| accessIPv4                           |                                                                  |
| accessIPv6                           |                                                                  |
| config_drive                         |                                                                  |
| created                              | 2016-12-23T11:54:36Z                                             |
| default network                      | 192.168.0.14                                                     |
| flavor                               | dc1.2x4.40 (196235bc-7ca5-4085-ac81-7e0242bda3f9)                |
| hostId                               | 001eaf9cf5cb5014ec21d10604c0b3140eb54c011a004812e930cbbc         |
| id                                   | f3ecfa36-be6a-4432-bacc-6bc9a7d0768e                             |
| image                                | Ubuntu 16.04 LTS (Xenial) (ef24441d-01ad-4006-b63d-6da67b7f1348) |
| key_name                             | datacentred                                                      |
| metadata                             | {}                                                               |
| name                                 | lxd0                                                             |
| os-extended-volumes:volumes_attached | []                                                               |
| progress                             | 0                                                                |
| security_groups                      | default, ssh_access                                              |
| status                               | ACTIVE                                                           |
| tenant_id                            | c71e35d5f6034d45aad211e6a7784b6d                                 |
| updated                              | 2016-12-23T11:54:45Z                                             |
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
(openstack)MacBook-Pro:DCOS matt$ nova floating-ip-associate f3ecfa36-be6a-4432-bacc-6bc9a7d0768e 185.98.151.85
```

Now we should be able to access our instance, so let's check the expected configuration for our network and LXD that we created using cloud-init is in place. Remember we need to use the keypair which we configured the instance with when we created it. 

```
MacBook-Pro:~ matt$ ssh -i ~/.ssh/mattj_dc ubuntu@185.98.151.85 'ifconfig'
br0       Link encap:Ethernet  HWaddr fa:16:3e:35:cc:c1  
          inet addr:192.168.0.14  Bcast:192.168.0.255  Mask:255.255.255.0
          inet6 addr: fe80::f816:3eff:fe35:ccc1/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:341 errors:0 dropped:0 overruns:0 frame:0
          TX packets:245 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:38610 (38.6 KB)  TX bytes:42387 (42.3 KB)

ens3      Link encap:Ethernet  HWaddr fa:16:3e:35:cc:c1  
          inet6 addr: fe80::f816:3eff:fe35:ccc1/64 Scope:Link
          UP BROADCAST RUNNING PROMISC MULTICAST  MTU:1500  Metric:1
          RX packets:357 errors:0 dropped:0 overruns:0 frame:0
          TX packets:252 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:44476 (44.4 KB)  TX bytes:42897 (42.8 KB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:188 errors:0 dropped:0 overruns:0 frame:0
          TX packets:188 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:13792 (13.7 KB)  TX bytes:13792 (13.7 KB)
```
```
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

When we want to add a host that DNSmasq will respond to, we just add a configuration line to our file /etc/dnsmasq.d/lxd.conf and reload DNSmasq :

```
MacBook-Pro:~ matt$ ssh -i ~/.ssh/mattj_dc ubuntu@185.98.151.85 "sudo sh -c 'echo dhcp-host=hugo,192.168.0.100 >> /etc/dnsmasq.d/lxd.conf' && sudo systemctl reload dnsmasq"
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
users:
  - default
  - name: hugo
packages:
  - hugo
  - git
runcmd:
  - su hugo - -c "git clone https://github.com/mattj-io/blog.git /home/hugo/mattj.blog"
  - su hugo - -c "git clone https://github.com/mattj-io/hugo-config.git /home/hugo/hugo-config"
  - ln -s /home/hugo/hugo-config/hugo.service /etc/systemd/system/hugo.service
  - su hugo - -c "cd /home/hugo/mattj.blog && git submodule init && git submodule update"
  - systemctl start hugo
EOF
) | lxc profile set hugo user.user-data -'
```

Basically what we're doing here is install the packages we need, using the packages module. 

We then pull down the website we want to serve from github, along with startup scripts for hugo, which we link into to place for systemd to use. 

Once all the configuration is in place, we can then use systemctl to start our service.

Before we can actually start our container, we have one more connectivity issue to sort out. The reason for this is that Neutron by default is configured for port level security. This takes the form of IPtables rules on the virtual router port of the instance, which only allow traffic with the MAC/IP combination of the virtual machine to pass through.

If your provider has the [port security extension](https://wiki.openstack.org/wiki/Neutron/ML2PortSecurityExtensionDriver) enabled, then you can turn off the port security features either at the port level or at the network level. My provider doesn't currently have this enabled, so I need a slightly more involved workaround.

In the default configuration, Neutron ports have the concept of allowed address pairs, which means we can add up to 10 entries which are allowed access across the port. IP addresses can be wildcarded in this configuration, but unfortunately not MAC addresses, so until my provider enables port security extension, I'm restricted to 10 containers per host.

Here's how we allow the container access to the port.

Firstly, we need to initialise the container so we can get it's MAC address

```
MacBook-Pro:~ matt$ ssh -i ~/.ssh/mattj_dc ubuntu@185.98.151.85 'lxc init ubuntu: hugo -p default -p hugo'
Creating hugo
MacBook-Pro:~ matt$ ssh -i ~/.ssh/mattj_dc ubuntu@185.98.151.85 'lxc config show hugo'
name: hugo
profiles:
- default
- hugo
config:
  volatile.apply_template: create
  volatile.base_image: fcc5bf9deb0a1f1ca924527feaf72f38b44ef5be48020820919c5270f2581cb4
  volatile.eth0.hwaddr: 00:16:3e:91:2c:9c
  volatile.last_state.idmap: '[{"Isuid":true,"Isgid":false,"Hostid":100000,"Nsid":0,"Maprange":65536},{"Isuid":false,"Isgid":true,"Hostid":100000,"Nsid":0,"Maprange":65536}]'
devices:
  root:
    path: /
    type: disk
ephemeral: false
```

Now we've got our MAC address, we can configure Neutron

First let's find the port we need, we're looking for our virtual machine IP address of 192.168.0.14 :

```
(openstack)MacBook-Pro:DCOS matt$ neutron port-list
+--------------------------------------+------------------------------------------+-------------------+--------------------------------------------------------------------------------------+
| id                                   | name                                     | mac_address       | fixed_ips                                                                            |
+--------------------------------------+------------------------------------------+-------------------+--------------------------------------------------------------------------------------+
| 9fae0b86-9f29-4709-a170-81ceb5a4a4d7 |                                          | fa:16:3e:11:bd:6d | {"subnet_id": "a2b72e5b-48ec-46f0-ab40-df56e698bea2", "ip_address": "192.168.0.1"}   |
| d2796309-92d0-418e-96c5-d255a72a2ec5 |                                          | fa:16:3e:4b:bd:86 | {"subnet_id": "a2b72e5b-48ec-46f0-ab40-df56e698bea2", "ip_address": "192.168.0.2"}   |
| d2cfaf77-b64c-4a70-afe6-bb3bb11dec2b |                                          | fa:16:3e:35:cc:c1 | {"subnet_id": "a2b72e5b-48ec-46f0-ab40-df56e698bea2", "ip_address": "192.168.0.14"}  |
+--------------------------------------+------------------------------------------+-------------------+--------------------------------------------------------------------------------------+
```

Now let's take a look at that port in more detail

```
(openstack)MacBook-Pro:DCOS matt$ neutron port-show d2cfaf77-b64c-4a70-afe6-bb3bb11dec2b
+-----------------------+-------------------------------------------------------------------------------------+
| Field                 | Value                                                                               |
+-----------------------+-------------------------------------------------------------------------------------+
| admin_state_up        | True                                                                                |
| allowed_address_pairs |                                                                                     |
| binding:vnic_type     | normal                                                                              |
| device_id             | f3ecfa36-be6a-4432-bacc-6bc9a7d0768e                                                |
| device_owner          | compute:None                                                                        |
| extra_dhcp_opts       |                                                                                     |
| fixed_ips             | {"subnet_id": "a2b72e5b-48ec-46f0-ab40-df56e698bea2", "ip_address": "192.168.0.14"} |
| id                    | d2cfaf77-b64c-4a70-afe6-bb3bb11dec2b                                                |
| mac_address           | fa:16:3e:35:cc:c1                                                                   |
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
(openstack)MacBook-Pro:DCOS matt$ neutron port-update d2cfaf77-b64c-4a70-afe6-bb3bb11dec2b --allowed-address-pairs type=dict list=true mac_address=00:16:3e:91:2c:9c,ip_address=192.168.0.0/24
Updated port: d2cfaf77-b64c-4a70-afe6-bb3bb11dec2b
```

Now when we look at that port again, we'll see that it's been updated to allow that combination of MAC and IP to use the port

```
(openstack)MacBook-Pro:DCOS matt$ neutron port-show d2cfaf77-b64c-4a70-afe6-bb3bb11dec2b
+-----------------------+-------------------------------------------------------------------------------------+
| Field                 | Value                                                                               |
+-----------------------+-------------------------------------------------------------------------------------+
| admin_state_up        | True                                                                                |
| allowed_address_pairs | {"ip_address": "192.168.0.0/24", "mac_address": "00:16:3e:91:2c:9c"}                |
| binding:vnic_type     | normal                                                                              |
| device_id             | f3ecfa36-be6a-4432-bacc-6bc9a7d0768e                                                |
| device_owner          | compute:None                                                                        |
| extra_dhcp_opts       |                                                                                     |
| fixed_ips             | {"subnet_id": "a2b72e5b-48ec-46f0-ab40-df56e698bea2", "ip_address": "192.168.0.14"} |
| id                    | d2cfaf77-b64c-4a70-afe6-bb3bb11dec2b                                                |
| mac_address           | fa:16:3e:35:cc:c1                                                                   |
| name                  |                                                                                     |
| network_id            | be9b8434-7ec2-46e3-9c5f-5759440a2b06                                                |
| security_groups       | 980deb5f-877b-460e-b11b-f0bfc6140931                                                |
|                       | e70972a7-0217-48f3-939c-7fdd3e914c75                                                |
| status                | ACTIVE                                                                              |
| tenant_id             | c71e35d5f6034d45aad211e6a7784b6d                                                    |
+-----------------------+-------------------------------------------------------------------------------------+
```

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
And if we test from our container host, our container has network access !

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

At this point, our cloud-init configuration will be being applied, configuring and starting the hugo service. 


The final piece of the puzzle here is how to get external network traffic in to our service, and for this we're going to leverage the Load Balancer as a Service functionality of OpenStack. My provider only currently support LBaaS v1, so that's what I'm going to use.

Firstly we need to create a load balancer pool :

```
(openstack)MacBook-Pro:DCOS matt$ neutron lb-pool-create --name web --protocol HTTP --subnet-id a2b72e5b-48ec-46f0-ab40-df56e698bea2 --lb-method ROUND_ROBIN
Created a new pool:
+------------------------+--------------------------------------+
| Field                  | Value                                |
+------------------------+--------------------------------------+
| admin_state_up         | True                                 |
| description            |                                      |
| health_monitors        |                                      |
| health_monitors_status |                                      |
| id                     | 695d37ef-4339-4d08-8ca7-ade826e50f2a |
| lb_method              | ROUND_ROBIN                          |
| members                |                                      |
| name                   | web                                  |
| protocol               | HTTP                                 |
| provider               | haproxy                              |
| status                 | PENDING_CREATE                       |
| status_description     |                                      |
| subnet_id              | a2b72e5b-48ec-46f0-ab40-df56e698bea2 |
| tenant_id              | c71e35d5f6034d45aad211e6a7784b6d     |
| vip_id                 |                                      |
+------------------------+--------------------------------------+
```

Now we have our pool, we need to assign it an IP on our internal network, and configure which ports and protocols we're using :

```
(openstack)MacBook-Pro:DCOS matt$ neutron lb-vip-create --address 192.168.0.200 --name web --protocol-port 80 --protocol HTTP --subnet-id a2b72e5b-48ec-46f0-ab40-df56e698bea2 web
Created a new vip:
+---------------------+--------------------------------------+
| Field               | Value                                |
+---------------------+--------------------------------------+
| address             | 192.168.0.200                        |
| admin_state_up      | True                                 |
| connection_limit    | -1                                   |
| description         |                                      |
| id                  | 2b106d9e-1d6a-43e6-974c-22dbef23d2b2 |
| name                | web                                  |
| pool_id             | 695d37ef-4339-4d08-8ca7-ade826e50f2a |
| port_id             | 802370ec-1dc3-463a-93eb-b2edb8102f63 |
| protocol            | HTTP                                 |
| protocol_port       | 80                                   |
| session_persistence |                                      |
| status              | PENDING_CREATE                       |
| status_description  |                                      |
| subnet_id           | a2b72e5b-48ec-46f0-ab40-df56e698bea2 |
| tenant_id           | c71e35d5f6034d45aad211e6a7784b6d     |
+---------------------+--------------------------------------+
```

Next we need to add our container as a member of the pool. My hugo configuration starts the server on port 8080, as I want to run it as an unprivileged user and we can't bind to port 80, so when we configure the load balancer we tell it we're listening on port 8080

```
(openstack)MacBook-Pro:DCOS matt$ neutron lb-member-create --address 192.168.0.100 --protocol-port 8080 web
Created a new member:
+--------------------+--------------------------------------+
| Field              | Value                                |
+--------------------+--------------------------------------+
| address            | 192.168.0.100                        |
| admin_state_up     | True                                 |
| id                 | f238f90f-f329-4380-b9be-3c7681d1f61d |
| pool_id            | 2e2c974c-65da-48ec-a849-783fe15b160e |
| protocol_port      | 8080                                 |
| status             | PENDING_CREATE                       |
| status_description |                                      |
| tenant_id          | c71e35d5f6034d45aad211e6a7784b6d     |
| weight             | 1                                    |
+--------------------+--------------------------------------+
```

Neutron load balancers can also have a health monitor service to handle failing over between members, but since we're only using a single member we don't need that function. 

The final part of our puzzle is to assign an external floating IP to the loadbalancer so we can access it from the internet. 

Let's find a free one from our pool first

```
(openstack)MacBook-Pro:DCOS matt$ neutron floatingip-list
+--------------------------------------+------------------+---------------------+--------------------------------------+
| id                                   | fixed_ip_address | floating_ip_address | port_id                              |
+--------------------------------------+------------------+---------------------+--------------------------------------+
| 8361ba16-8088-47f5-bf52-e7929a0dd4f3 | 192.168.0.14     | 185.98.151.85       | d2cfaf77-b64c-4a70-afe6-bb3bb11dec2b |
| 87b3965a-7c88-405e-9034-b93bf9e10009 |                  | 185.98.151.139      |                                      |
| c68002f5-bf63-460f-be3b-f15fb80064b0 |                  | 185.98.151.108      |                                      |
+--------------------------------------+------------------+---------------------+--------------------------------------+
```

We need to attach the floating IP directly to the neutron port of our loadbalancer, so we also need the port ID. 

```
(openstack)MacBook-Pro:DCOS matt$ neutron port-list
+--------------------------------------+------------------------------------------+-------------------+--------------------------------------------------------------------------------------+
| id                                   | name                                     | mac_address       | fixed_ips                                                                            |
+--------------------------------------+------------------------------------------+-------------------+--------------------------------------------------------------------------------------+
| 7db2c0f3-f7bc-491d-bacc-951dc189b91b | vip-3b2db384-f929-4860-8897-645ab3cd1cba | fa:16:3e:db:10:73 | {"subnet_id": "a2b72e5b-48ec-46f0-ab40-df56e698bea2", "ip_address": "192.168.0.200"} |
| 9fae0b86-9f29-4709-a170-81ceb5a4a4d7 |                                          | fa:16:3e:11:bd:6d | {"subnet_id": "a2b72e5b-48ec-46f0-ab40-df56e698bea2", "ip_address": "192.168.0.1"}   |
| d2796309-92d0-418e-96c5-d255a72a2ec5 |                                          | fa:16:3e:4b:bd:86 | {"subnet_id": "a2b72e5b-48ec-46f0-ab40-df56e698bea2", "ip_address": "192.168.0.2"}   |
| d2cfaf77-b64c-4a70-afe6-bb3bb11dec2b |                                          | fa:16:3e:35:cc:c1 | {"subnet_id": "a2b72e5b-48ec-46f0-ab40-df56e698bea2", "ip_address": "192.168.0.14"}  |
+--------------------------------------+------------------------------------------+-------------------+--------------------------------------------------------------------------------------+
```

With all that information we can now associate the floating IP to the port :

```
(openstack)MacBook-Pro:DCOS matt$ neutron floatingip-associate c68002f5-bf63-460f-be3b-f15fb80064b0 7db2c0f3-f7bc-491d-bacc-951dc189b91b
Associated floating IP c68002f5-bf63-460f-be3b-f15fb80064b0
```

We also need to remember that, as with instances, load balancer ports have security groups, and by default no access is allowed. 

As we did at the start to allow ssh access, let's create another one for web access 

```
(openstack)MacBook-Pro:DCOS matt$ nova secgroup-create web Web
+--------------------------------------+------+-------------+
| Id                                   | Name | Description |
+--------------------------------------+------+-------------+
| ffae1901-1219-4e7c-bf16-9e0f2d51dd5f | web  | Web         |
+--------------------------------------+------+-------------+
```

Now we need to add a rule to the security group for web access. 

```
(openstack)MacBook-Pro:DCOS matt$ nova secgroup-add-rule web tcp 80 80 0.0.0.0/0
+-------------+-----------+---------+-----------+--------------+
| IP Protocol | From Port | To Port | IP Range  | Source Group |
+-------------+-----------+---------+-----------+--------------+
| tcp         | 80        | 80      | 0.0.0.0/0 |              |
+-------------+-----------+---------+-----------+--------------+
```

And finally we add that security group to the port

```
(openstack)MacBook-Pro:DCOS matt$ neutron port-update 7db2c0f3-f7bc-491d-bacc-951dc189b91b --security-group web
Updated port: 7db2c0f3-f7bc-491d-bacc-951dc189b91b
```

Now when we point a browser at our external floating IP, we reach this website !

So what I've outlined in these two blog posts is a fully automatable process for creating and managing LXD containers in a standard OpenStack environment, using only native OpenStack and cloud-init tooling. Although I haven't tied the pieces together, hopefully it's clear that it would be trivial to do that in your language of choice given that every element can be deployed and configured without manual intervention. 

Bear in mind this is a very simplistic approach and only works well for self-contained services. For this to work in more complex environments we would ideally integrate this with a discovery and orchestration toolset, perhaps based on gossip protocols like those used by [Consul/Serf](https://www.consul.io/).  
