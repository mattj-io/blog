+++
title = "Nova Rescue"
description = "Using the rescue functionality in Nova"
date = "2015-10-22T07:56:59Z"

+++

Occasionally you may have a situation where a virtual machine won't boot correctly, or you cannot get into it. 

It's not very well known, but OpenStack provides a mechanism to rescue virtual instances in this state and repair them.

Firstly boot the instance into rescue mode. You'll need the instance ID in order to do this, which you can find in Horizon or via the CLI.

```
nova rescue $instance_id
```

You'll see in Horizon and the CLI tools that the instance is now showing as having status Rescue. 

```
nova show $instance_id
 
---- output cut ---
| name                                 | 14.04LTSbuild                                            |
 
| net1 network                         | 192.168.0.4, 185.43.218.143                              |
 
| os-extended-volumes:volumes_attached | []                                                       |
 
| security_groups                      | default                                                  |
 
| status                               | RESCUE 
 
---- output cut ----
```

What will now happen is that the instance will be booted with a special rescue disk, and the original disk will be attached to the instance as /dev/vdb. You then need to log into the instance, using the key you configured from the original instance, and mount the drive somewhere.

```
mkdir /mnt/rescue
mount /dev/vdb1 /mnt/rescue
```

You can now explore your filesystem in /mnt/rescue, edit whatever files need to be fixed and then umount the disk

```
umount /mnt/rescue
```

Now log out, and put the instance back into normal mode. This will restart the instance with the original disk attached again. 

```
nova unrescue $instance_id
```

Providing you've managed to fix the original issue, your instance should now be available again !
