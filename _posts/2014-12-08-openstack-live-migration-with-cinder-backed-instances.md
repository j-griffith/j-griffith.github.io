---
layout: post
title: OpenStack Live Migration With Cinder Backed Instances
comments: true
---
<meta http-equiv='Content-Type' content='text/html; charset=utf-8' />

Over the last several weeks I keep getting questions about Live Migration, does it work, how do you configure it, what are the requirements to use it and so on.  To be honest up until recently I hadn’t actually dove into Live Migration so I tried it out (and failed), then started asking around and resorting to Google Searches.  The good thing is, there’s TONS of information out there, problem is, there’s TONS of information out there.  The bad thing here is the info isn’t all that consistent.  Like many things there’s always more than one way to achieve an end result, Live Migration is really no different.  So I thought I’d try and write up what worked for me and in my opinion was the simplest method to use.

First off, for those that aren’t familiar; Live Migration of OpenStack Instances is the process of migrating an OpenStack Instance from one Compute Node to another with little or no downtime.  The docs page has a good description here: [OpenStack Admin-Guide](http://docs.openstack.org/admin-guide-cloud/content/section_configuring-compute-migrations.html)

One of the big misconceptions about Live Migration is that if you’re Instances aren’t stored on a Shared FS or a Shared backend device like CEPH you can’t do Live-Migration.  That’s not true at all, in fact notice the doc has an entry specifically for Volume-Backed Instances.  So let’s take a look at how to create a Volume-Backed Instance using Juno and migrate it from one Compute Host to another.

First, my config; I’ve set up a multi-node deployment using devstack stable/juno.  I went ahead and configured Cinder to run multiple backends, in my case SolidFire and LVM because I wanted to test both and make sure everything worked properly.  For info on Cinder Multi-Backend checkout the wiki here: [Cinder multi-backend](https://wiki.openstack.org/wiki/Cinder-multi-backend)

I chose to specify ssh as my connection method for live-migration.  There are of course other ways to do this by configuring libvirt and qemu, which you can check out here: [libvirt config](https://libvirt.org/uri.html) but rather than do any of that I just wanted to put a simple entry in my nova.conf file and use ssh.  Here’s the libvirt section of my nova.conf:

```
[libvirt]
inject_partition = -2
use_usb_tablet = False
cpu_mode = none
virt_type = kvm
live_migration_uri = qemu+ssh://root@%s/system
```

Notice I’m using root here in my connection.  I’ve used “regular” accounts with sudo permissions before to do remote connections to libvirt, but there’s something about the process from Nova that the Key Verification will fail for any user other than ‘root’.  If anybody knows how to fix this up let me know, It’d be great to learn what I’m missing here.  For those that are interested in more detail, my nova.conf other than the entry here is just the defaults from devstack.  Here’s a link to my nova.conf on the controller/compute: /etc/nova/nova.conf.  I’ve also added a link to my devstack local.conf; in this setup I actually started with a SolidFire deployment and manually updated cinder.conf to do multi-backend with LVM and added the default volume-type: devstack local.conf

So since we’re using ssh and root, you’ve probably guessed we’re going to need to setup ssh keys between our compute nodes.  I went ahead and setup root keys on each of the compute nodes, and installed each machines key on all of the nodes just doing a simple ssh-copy-id.  You may want to get a bit more sophisticated, distribute a key and modify /root/.ssh/config to include entries for each of the compute nodes.  Something like this should do the trick:

```
Host fluffy-compute
HostName fluffy-compute
User root
IdentityFile ~/.ssh/live-migrationkey.pem
```

Verify you can ssh as root to->from each of the Compute nodes, restart Nova Compute services and you *should* be ready to go.  Notice there I said *should*, what i found out was things didn’t work.  I’d get an error message stating that I couldn’t do live migration because I wasn’t using shared storage… huh?  What’s up with that?

So it turns out there was a bug introduced during the Juno development cycle.  You can check out the bug on LaunchPad here:[LP-Bug]( https://bugs.launchpad.net/nova/+bug/1392773)

Fortunately there’s a proposal up for master already that can be pulled down (and hopefully will merge soon), but I was running Juno, so I put together a quick backport to try out.  Note this path isn’t quite complete, needs some method signature updates for the other hypervisors in OpenStack and of course Unit Tests.  Anyway, you can check out what I started via this [gist](https://gist.github.com/j-griffith/5e9be4e6548a03697dc5)

I went ahead an applied the Juno patch from the gist above, restarted all of my Nova services and tried again.  Here’s how things go:

First, grab the image-id you want to use and create a Cinder Volume with it…

```
ubuntu@fluffy-master:~$ nova image-list
+--------------------------------------+---------------------------------+--------+--------+
| ID                                   | Name                            | Status | Server |
+--------------------------------------+---------------------------------+--------+--------+
| 40520350-25c7-4786-9586-5430d52d7663 | RedHat 6.5 Server               | ACTIVE |        |
| 8a69ae88-a729-4d68-a7e6-25048b1cf885 | RedHat 7.0 Server               | ACTIVE |        |
| afaa2feb-3a58-4e72-bebd-38f66bd0b611 | Ubuntu 14.04 Server             | ACTIVE |        |
| 28b77f27-a922-4627-8b1d-f0df627be907 | cirros-0.3.2-x86_64-uec         | ACTIVE |        |
| 63c173cd-3f5a-49ff-89ab-8c90a1701808 | cirros-0.3.2-x86_64-uec-kernel  | ACTIVE |        |
| e3aa2ab0-383c-4f75-bdc0-dbee86645ad5 | cirros-0.3.2-x86_64-uec-ramdisk | ACTIVE |        |
+--------------------------------------+---------------------------------+--------+--------+
```

```
ubuntu@fluffy-master:~$ cinder create --image-id \
afaa2feb-3a58-4e72-bebd-38f66bd0b611 --display-name trusty-volume 10
+---------------------------------------+--------------------------------------+
| Property                              | Value |
+---------------------------------------+--------------------------------------+
| attachments                           | []                                   |
| availability_zone                     | nova                                 |
| bootable                              | false                                |
| consistencygroup_id                   | None                                 |
| created_at                            | 2014-12-08T17:34:50.000000           |
| description                           | None                                 |
| encrypted                             | False                                |
| id                                    | 6419112b-bf40-4994-809f-12dce25f5f1f |
| metadata                              | {}                                   |
| name                                  | trusty-volume                        |
| os-vol-host-attr:host                 | None                                 |
| os-vol-mig-status-attr:migstat        | None                                 |
| os-vol-mig-status-attr:name_id        | None                                 |
| os-vol-tenant-attr:tenant_id          | 56dc791650074cc9a2858bd5053a7116     |
| os-volume-replication:driver_data     | None                                 |
| os-volume-replication:extended_status | None                                 |
| replication_status                    | disabled                             |
| size                                  | 10                                   |
| snapshot_id                           | None                                 |
| source_volid                          | None                                 |
| status                                | creating                             |
| user_id                               | 3dcc1572d7594a4eb6b043b819e415de     |
| volume_type                           | solidfire                            |
+---------------------------------------+--------------------------------------+
```

```
ubuntu@fluffy-master:~$cinder list
+--------------------------------------+-----------+---------------+------+-------------+----------+-------------+
| ID                                   | Status    | Name          | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+---------------+------+-------------+----------+-------------+
| 6419112b-bf40-4994-809f-12dce25f5f1f | available | trusty-volume | 10   | solidfire   | true     |             |
+--------------------------------------+-----------+---------------+------+-------------+----------+-------------+
```

```
ubuntu@fluffy-master:~$ nova boot --flavor 2 --block-device-mapping \
vda=6419112b-bf40-4994-809f-12dce25f5f1f --key-name fluffycloud trusty-migrate
+--------------------------------------+--------------------------------------------------+
| Property                             | Value                                            |
+--------------------------------------+--------------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                           |
| OS-EXT-AZ:availability_zone          | nova                                             |
| OS-EXT-SRV-ATTR:host                 | -                                                |
| OS-EXT-SRV-ATTR:hypervisor_hostname  | -                                                |
| OS-EXT-SRV-ATTR:instance_name        | instance-00000002                                |
| OS-EXT-STS:power_state               | 0                                                |
| OS-EXT-STS:task_state                | scheduling                                       |
| OS-EXT-STS:vm_state                  | building                                         |
| OS-SRV-USG:launched_at               | -                                                |
| OS-SRV-USG:terminated_at             | -                                                |
| accessIPv4                           |                                                  |
| accessIPv6                           |                                                  |
| adminPass                            | rKL9sfs2vPs4                                     |
| config_drive                         |                                                  |
| created                              | 2014-12-08T17:41:39Z                             |
| flavor                               | m1.small (2)                                     |
| hostId                               |                                                  |
| id                                   | 9afb79e1-f07a-4c98-9d6c-9ce66dcbf904             |
| image                                | Attempt to boot from volume - no image supplied  |
| key_name                             | fluffycloud                                      |
| metadata                             | {}                                               |
| name                                 | trusty-migrate                                   |
| os-extended-volumes:volumes_attached | [{6419112b-bf40-4994-809f-12dce25f5f1f&quot;}]   |
| progress                             | 0                                                |
| security_groups                      | default                                          |
| status                               | BUILD                                            |
| tenant_id                            | 56dc791650074cc9a2858bd5053a7116                 |
| updated                              | 2014-12-08T17:41:39Z                             |
| user_id                              | 3dcc1572d7594a4eb6b043b819e415de                 |
+--------------------------------------+--------------------------------------------------+
```

```
ubuntu@fluffy-master:~$ nova list
+--------------------------------------+----------------+--------+------------+-------------+------------------+
| ID                                   | Name           | Status | Task State | Power State | Networks         |
+--------------------------------------+----------------+--------+------------+-------------+------------------+
| 9afb79e1-f07a-4c98-9d6c-9ce66dcbf904 | trusty-migrate | ACTIVE | -          | Running     | private=10.1.0.2 |
+--------------------------------------+----------------+--------+------------+-------------+------------------+
```

```
ubuntu@fluffy-master:~$ nova show 9afb79e1-f07a-4c98-9d6c-9ce66dcbf904
+--------------------------------------+----------------------------------------------------------+
| Property                             | Value                                                    |
+--------------------------------------+----------------------------------------------------------+
| OS-DCF:diskConfig                    | MANUAL                                                   |
| OS-EXT-AZ:availability_zone          | nova                                                     |
| OS-EXT-SRV-ATTR:host                 | fluffy-compute                                           |
| OS-EXT-SRV-ATTR:hypervisor_hostname  | fluffy-compute                                           |
| OS-EXT-SRV-ATTR:instance_name        | instance-00000002                                        |
| OS-EXT-STS:power_state               | 1                                                        |
| OS-EXT-STS:task_state                | -                                                        |
| OS-EXT-STS:vm_state                  | active                                                   |
| OS-SRV-USG:launched_at               | 2014-12-08T17:41:47.000000                               |
| OS-SRV-USG:terminated_at             | -                                                        |
| accessIPv4                           |                                                          |
| accessIPv6                           |                                                          |
| config_drive                         |                                                          |
| created                              | 2014-12-08T17:41:39Z                                     |
| flavor                               | m1.small (2)                                             |
| hostId                               | 48adbabbab9babab4c8948d3478e9d2fae8ffb407bab1ca21a843981 |
| id                                   | 9afb79e1-f07a-4c98-9d6c-9ce66dcbf904                     |
| image                                | Attempt to boot from volume - no image supplied          |
| key_name                             | fluffycloud                                              |
| metadata                             | {}                                                       |
| name                                 | trusty-migrate                                           |
| os-extended-volumes:volumes_attached | [{6419112b-bf40-4994-809f-12dce25f5f1f;}]                |
| private network                      | 10.1.0.2                                                 |
| progress                             | 0                                                        |
| security_groups                      | default                                                  |
| status                               | ACTIVE                                                   |
| tenant_id                            | 56dc791650074cc9a2858bd5053a7116                         |
| updated                              | 2014-12-08T17:41:47Z                                     |
| user_id                              | 3dcc1572d7594a4eb6b043b819e415de                         |
+--------------------------------------+----------------------------------------------------------+
```

```
    ubuntu@fluffy-master:~$ubuntu@fluffy-master:~$ nova live-migration \
	9afb79e1-f07a-4c98-9d6c-9ce66dcbf904
```

```
    ubuntu@fluffy-master:~$ nova list
    +--------------------------------------+----------------+-----------+------------+-------------+------------------+
    | ID | Name | Status | Task State | Power State | Networks |
    +--------------------------------------+----------------+-----------+------------+-------------+------------------+
    | 9afb79e1-f07a-4c98-9d6c-9ce66dcbf904 | trusty-migrate | MIGRATING | migrating | Running | private=10.1.0.2 |
    +--------------------------------------+----------------+-----------+------------+-------------+------------------+
```

```
    ubuntu@fluffy-master:~$nova list
    +--------------------------------------+----------------+--------+------------+-------------+------------------+
    | ID | Name | Status | Task State | Power State | Networks |
    +--------------------------------------+----------------+--------+------------+-------------+------------------+
    | 9afb79e1-f07a-4c98-9d6c-9ce66dcbf904 | trusty-migrate | ACTIVE | - | Running | private=10.1.0.2 |
    +--------------------------------------+----------------+--------+------------+-------------+------------------+
```

```
    ubuntu@fluffy-master:~$ nova show 9afb79e1-f07a-4c98-9d6c-9ce66dcbf904
    +--------------------------------------+----------------------------------------------------------+
    | Property                             | Value                                                    |
    +--------------------------------------+----------------------------------------------------------+
    | OS-DCF:diskConfig                    | MANUAL                                                   |
    | OS-EXT-AZ:availability_zone          | nova                                                     |
    | OS-EXT-SRV-ATTR:host                 | fluffy-master                                            |
    | OS-EXT-SRV-ATTR:hypervisor_hostname  | fluffy-master                                            |
    | OS-EXT-SRV-ATTR:instance_name        | instance-00000002                                        |
    | OS-EXT-STS:power_state               | 1                                                        |
    | OS-EXT-STS:task_state                | -                                                        |
    | OS-EXT-STS:vm_state                  | active                                                   |
    | OS-SRV-USG:launched_at               | 2014-12-08T17:41:47.000000                               |
    | OS-SRV-USG:terminated_at             | -                                                        |
    | accessIPv4                           |                                                          |
    | accessIPv6                           |                                                          |
    | config_drive                         |                                                          |
    | created                              | 2014-12-08T17:41:39Z                                     |
    | flavor                               | m1.small (2)                                             |
    | hostId                               | 97a1a99eaf8e1fab2ea7cb0be641ff55f2da5df4e68d0ecb62293e00 |
    | id                                   | 9afb79e1-f07a-4c98-9d6c-9ce66dcbf904                     |
    | image                                | Attempt to boot from volume - no image supplied          |
    | key_name                             | fluffycloud                                              |
    | metadata                             | {}                                                       |
    | name                                 | trusty-migrate                                           |
    | os-extended-volumes:volumes_attached | [{6419112b-bf40-4994-809f-12dce25f5f1f}]                 |
    | private network                      | 10.1.0.2                                                 |
    | progress                             | 0                                                        |
    | security_groups                      | default                                                  |
    | status                               | ACTIVE                                                   |
    | tenant_id                            | 56dc791650074cc9a2858bd5053a7116                         |
    | updated                              | 2014-12-08T17:41:47Z                                     |
    | user_id                              | 3dcc1572d7594a4eb6b043b819e415de                         |
    +--------------------------------------+----------------------------------------------------------+
```

Pretty cool eh?  As always feedback is more than welcome.  Also, if you have specific OpenStack / Cinder related topics you might want to see more about, drop me a line or post a comment.

Thanks!!!



