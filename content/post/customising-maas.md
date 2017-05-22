+++
title = "Customising MAAS installs"
description = "Customising installs using MAAS"
date = "2017-05-22T14:56:39+01:00"

+++
Canonical's [MAAS](https://maas.io) is a bare metal provisioning and lifecycle management system. In this post, we'll look at customising installs provisioned by MAAS to enable site specific configuration. 

Historically, the paradigm for bare metal machine installation has been to use a ‘golden master’ image, which contains in it all of the site specific customisation required for operations. This brings with it a lot of complexity around maintenance, since the images are fixed. Updating requires building entirely new images each time, and normally requires maintaining libraries of brittle shell scripts to do the customisation and build the image. 

In the cloud era new mechanisms have evolved to do post-installation configuration, allowing for much greater flexibility and ease of maintenance in the workflows around deployment. Images made for clouds, which include the images that MAAS uses, generally use the [cloud-init](https://https://cloudinit.readthedocs.io/en/latest/)  mechanism developed by Canonical in order to do post deployment configuration to allow for highly flexible early installation configuration. 

In general in cloud environments, best practice is to pass over to a configuration management or orchestration system as soon as possible post installation, so to do just as much as necessary during cloud-init to enable this.

## MAAS installs

MAAS’s internal deployment mechanism uses [curtin](https://curtin.readthedocs.io/en/latest/index.html) (short for Curt Installer), written by Canonical to handle multiple operating system installation, and based on copying an image file to the hard disk and then leveraging cloud-init for final configuration on first boot. The image MAAS copies therefore has to be in a specific format and include the client side elements of cloud-init.  

When cloud-init starts up the first time after a MAAS controlled install, it queries the MAAS data source at the region controller IP address and downloads a set of data to customise the image with elements like hostname, keys and network configuration. The final stage of the cloud-init process is to include userdata, and it is at this point that you have the opportunity to customise the data provided to the instance to make additional customisations.

## Workflows and Best Practice

In circumstances where centralised configuration management or orchestration is not in use, it is entirely possible to run complex command sets directly from the cloud init userdata in order to achieve customisation of the booted image. However, a more streamlined workflow would be to use the userdata to install a configuration management tool, pull configuration management code from a repository, and run the code locally on the target. This enables the deployment customisation to be held in a version control system, which aids configuration maintenance and auditability. 

## Userdata Layout

The userdata which curtin uses is located in the etc/maas/preseeds/ directory on the MAAS Region controller. When curtin loads the curtin_userdata file it will pick the one that is most specific to the machine and OS it is starting based on the following priority:

```
{prefix}_{osystem}_{node_arch}_{node_subarch}_{release}_{node_name}
{prefix}_{osystem}_{node_arch}_{node_subarch}_{release}
{prefix}_{osystem}_{node_arch}_{node_subarch}
{prefix}_{osystem}_{node_arch}
{prefix}_{osystem}
{prefix}
'Generic'
```

All userdata files use the same prefix ‘curtin_userdata’. You can see the other elements that MAAS uses to refer to images in the folder structure where MAAS stores images - /var/lib/boot-resources/current. 

So, for example, for CentOS, we can see MAAS uses the folder structure:

```
/var/lib/maas/boot-resources/current/centos/amd64/generic/centos70/
```

Therefore if we want to create custom userdata that will apply to all CentOS 7 installs on x86_64, we need to create a file in /etc/maas/preseeds called :

```
curtin_userdata_centos_amd64_generic_centos70
```

The easiest way to do this is to copy the generic centos userdata file and use that as a template. Make sure the ownership and permissions are the same as the other files in the directory. 

```
cd /etc/maas/preseeds
cp curtin_userdata_centos curtin_userdata_centos_amd64_generic_centos70
```

## Curtin Syntax

Curtin has a special cloud-init syntax for running commands inside the install image, which we need to use since during the install process we’re actually running in a RAM disk. We need to call curtin like this :

```
curtin --in-target -- $CMD
```

To ensure the shell gets the appropriate environment variables, shell commands ideally get called like this :

```
curtin --in-target -- sh -c “$CMD”
```

In the actual userdata file, the section we’re interested in is the late_commands section, which get run at the end of the install process. Each command in the section needs an arbitrary unique name, and if you need to enforce ordering, you can use numbering in the name as is shown in the example code.  

The example below illustrates an example using Ansible, and shows the syntax needed for the commands. 

```
late_commands:
  maas: [wget, '--no-proxy', '{{node_disable_pxe_url}}', '--post-data', '{{node_disable_pxe_data}}', '-O', '/dev/null']
  10_epel: ["curtin", "in-target", "--", "sh", "-c", "/usr/bin/yum -y install epel-release"]
  20_ansible: ["curtin", "in-target", "--", "sh", "-c", "/usr/bin/yum -y install ansible"]
  30_git: ["curtin", "in-target", "--", "sh", "-c", "/usr/bin/yum -y install git"]
  40_clone: ["curtin", "in-target", "--", "sh", "-c", "git clone https://github.com/ansible/ansible-examples.git /srv/ansible-examples"]
  50_runansible[“curtin”, “in-target”, “--”, “sh”, “-c”, “ansible-playbook -i /srv/ansible-examples/hosts /srv/ansible-examples/example_playbook.yml”]
```

This code will enable the Epel repository, install ansible from that repository, install git, clone an example ansible repository into /srv, then run a playbook from that repository. 

This example provides a maintainable solution to image customisation where centralised orchestration or configuration management is not in use, without having to maintain ‘golden master’ images, and customisation in MAAS is kept to a minimum.


