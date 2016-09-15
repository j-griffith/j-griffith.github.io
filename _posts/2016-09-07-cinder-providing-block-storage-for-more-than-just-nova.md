---
layout: post
title: Cinder - Block Storage for things other than Nova
comments: true
---

<meta http-equiv='Content-Type' content='text/html; charset=utf-8' />
# Cinder as storage for Docker (part 1 of 2)

A lot of you may be familiar with Cinder, the OpenStack Block Storage project that's been around for quite a while now.  If you're not, you should take a look at the series of blog posts my friend [Ken Hui](https://cloudarchitectmusings.com/) at Rackspace put together, [Laying Cinder Block](http://blog.rackspace.com/laying-cinder-block-volumes-in-openstack-part-1-the-basics).  That's a more detailed version of the presentation he and I gave at the Summit in Atlanta a couple years back.

The context most think of when they consider Cinder is always "Block Storage for Nova Instances".  The fact is however that it's pretty useful outside of the traditional Nova context.  You can use Cinder as a powerful "Block Storage as a Service" resource for a number of use cases and environments.  Myself and a few other have been doing this for a number of years now, mostly with Bare Metal systems in our labs.

Given that Containers are the hottest thing since sliced bread these days, I thought it would be fun to post a little how to on how to use Cinder by itself with Containers.  For those of you that saw my presentation at the OpenStack Summit in Austin last spring, here's what I was trying to show you on stage when I lost my wifi connection.

On that note, here's part 1 of a 2 part posting about using Cinder a little bit differently than you may have been doing previously.  For those that were at OpenStack Days East this is a follow up to the presentation I gave there, those [slides](2016-09-07-cinder-providing-block-storage-for-more-than-just-nova.md) are also available for those that missed it.

## A neat use case that works for me

So one pattern that I've found myself using more and more that's kind of interesting is that my development is a bit mixed.  Some times I'm doing things on an OpenStack Instance, other times on a workstation or bare metal server, and more recently working in Containers (ok, and sometimes public cloud... but we'll let the one slide for now).

What I found was that I could use my Cinder deployment (by itself, or as part of a full OpenStack deployment) to manage my storage needs across all three models fairly easily.  Even better, I could start working on something in one environment, and easily take my data with me to another.  I've now dones this with some build images and data base files not just from my workstation to a Nova Instance, but also now to Docker Containers.  As I change my app to run in various environments I just unplug the storage and plug it back in where I want it.  Kinda cool right?

## Cinder doesn't know or care who's consuming it (mostly)

The biggest point of all of this is that for the most part, Cinder doesn't know (or care) who or what is calling it's API or even really what they're doing with the resources.  It's mostly just a control plane to bridge the differences between various backend storage devices.  What a caller chooses to do with those resources is up to them, and it doesn't really matter if that's a person using the respponse data to type in info on a cmd line, an automation script on a bare metal server, or in our case today a Docker Volume Plugin.

I will reluctantly point out that over the years there have been some things that have crept into the code that make Cinder a bit more Nova aware (which is unfortunate) but it's still pretty easy to work around.  Most of it is just things like requiring Instance UUID's for attach (that's a failry generic thing though) and also there are some backends that have snuck in some *features* that use Nova to help them out.  In this case today I'm focused on just using the LVM driver and iSCSI connectivity.  This also works for other iSCSI based backends including SolidFire and others.

## A Cinder plugin for Docker

Containers are hot, no doubt about it.  They're useful, the provide some pretty awesome flexibility in terms of not only development, but more importantly are fanstastic when it comes to deployment.  So, while I've been using Cinder against my Bare Metal systems and OpenStack Cloud for a while, last fall I started doing more and more with Containers, so I decided it would be worthwhile to write a [Cinder Plugin for Docker](https://github.com/j-griffith/cinder-docker-driver) that would allow me to use Cinder directly.

### Yes, this has been done before

So the first thing people pointed out was that "you can do this already".  Yup, that's true (ish), there are a number of offerings in the Container space that have created plugins for Docker that run on top of and consume Cinder.  Some fo them even work (some of them don't).  None of them have much involvement from the Cinder community, and none of them are completely Cinder focused.

Anyway, that's all great, if you use one of those (or choose to use one in the future) good for you!  That's awesome, but if you are looking for something that is "just" Cinder and is not maintained by a storage vendor maybe this will be of interest to you.

*Disclosure* Yes, I work for SolidFire/Netapp.  There's no underlying interest here for me though other than Cinder and cool tech.  If you know me from the community I'm pretty good at keeping company and community interests seperate.  I'm also fortunate enough to work for some great folks that *get it* and understand the importance.

### Docker Volume Plugins

Docker started offering a Volume API a few releases back, it's actually really pretty simple.  You can read up on it a bit [here](https://docs.docker.com/engine/extend/plugins_volume/) if you're interested.

The key here is that it's pretty simple, and that's a good... no that's a *GREAT* thing.  You'll notice if you poke around on that link that there are quite a few volume plugins available.  You also might notice that several of the devices listed out there also have Cinder plugins/drivers.

## The Cinder plugin
All of the plugins are pretty similar in how they work, the majority of us choose to use golang and the now official Docker Plugins Helper module.  Between that and the awesome [Gophercloud](https://github.com/rackspace/gophercloud) golang bindings for OpenStack started by RackSpace, creating the plugin was pretty easy.  The only effort here was the addition of some code to do the attach/detach of the Cinder volumes, even that though is still pretty trivial.  I just added some open-iscsi wrappers to the plugin and voila.

*warning* I've only implemented iSCSI.. because; well it's my favorite.  Those that offer other options are more than welcome to submit PR's if they're interested.

Here are some of the reasons for creating this plugin rather than just pointing to and using someone elses:
1. I happen to consider myself a bit of a Cinder expert
2. Vendor agnostic (my Github repo is mine)
3. I'm focused on just Cinder that's it (other variants have other motivations)
4. Licenesed under the Unlicense so you don't have to be afraid to contribute, sign a EULA etc.  (but please don't start submitting stuff with copyright headers etc)

###  Let's get started
##### Pre Requisites

For Ubuntu:

    sudo apt-get install open-iscsi

For RHEL variants:

    sudo yum install iscsi-initiator-utils

Of course make sure you have Docker installed, I recommend 1.12 at this point.

Currently the plugin is a simple daemon written in Golang.  You'll need to isntall it on your Docker Node(s), edit a simple config file to point it to your Cinder Endpoint and fire it up.  I've recently created a systemd init service for this that the install script will create and set everything up for you.  So all you have to do is use curl, or wget and pull down the install script and run it as root (or with sudo)

```
curl -sSL https://raw.githubusercontent.com/j-griffith/cinder-docker-driver/master/install.sh \
| sudo sh
```

Next we just set up a minimal config file so we can talk to our Cinder Endpoint (standard OpenStack stuff here), we'll just create a json file with the needed credential info ```/var/lib/cinder/dockerdriver/config.json```:

```
    {
        "HostUUID": "82ee38eb-821b-42d0-8c61-c4974d0c8536",
        "Endpoint": "http://172.16.140.243:5000/v2.0",
        "Username": "jdg",
        "Password": "NotMyPassword",
        "TenantID": "3dce5dd10b414ac1b942aba8ce8558e7"
    }
```
Now, if you've used the install script you can just start the service:
```sudo service cinder-docker-driver restart```

Verify everything came up properly:

```
$ sudo service cinder-docker-driver status
● cinder-docker-driver.service - "Cinder Docker Plugin daemon"
   Loaded: loaded (/etc/systemd/system/cinder-docker-driver.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2016-09-08 23:44:06 UTC; 29min ago
 Main PID: 13917 (cinder-docker-d)
    Tasks: 6
   Memory: 5.3M
      CPU: 290ms
   CGroup: /system.slice/cinder-docker-driver.service
           └─13917 /usr/bin/cinder-docker-driver &

Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=debug msg="getVolByName: `cvol-4`"
Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=debug msg="querying volume: {Attachments:[] AvailabilityZone:nova Bootable:false ConsistencyGroupID: CreatedAt:2016-09-08T23:46:13.000000 Description:Docker volume. Encrypted:false Name:cvol-4 VolumeType:lvmdriver-1 ReplicationDriverData: ReplicationE
Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=debug msg="Found Volume ID: 677a821b-e33b-44b5-8fb3-38315e832216"
Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=info msg="Remove/Delete Volume: cvol-4"
Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=debug msg="getVolByName: `cvol-4`"
Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=debug msg="querying volume: {Attachments:[] AvailabilityZone:nova Bootable:false ConsistencyGroupID: CreatedAt:2016-09-08T23:46:13.000000 Description:Docker volume. Encrypted:false Name:cvol-4 VolumeType:lvmdriver-1 ReplicationDriverData: ReplicationE
Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=debug msg="Found Volume ID: 677a821b-e33b-44b5-8fb3-38315e832216"
Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=debug msg="Remove/Delete Volume ID: 677a821b-e33b-44b5-8fb3-38315e832216"
Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=debug msg="Response from Delete: {ErrResult:{Result:{Body:<nil> Header:map[] Err:<nil>}}}\n"
Sep 08 23:47:07 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:47:07Z" level=info msg="List volumes: "
lines 1-20/20 (END)
```
Then you can use journalctl to filter out the relative entries in syslog:

``` bash
ubuntu@box-1:~$ journalctl -f -u cinder-docker-driver
-- Logs begin at Thu 2016-09-08 20:52:15 UTC. --
Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=debug msg="getVolByName: `cvol-4`"
Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=debug msg="querying volume: {Attachments:[] AvailabilityZone:nova Bootable:false ConsistencyGroupID: CreatedAt:2016-09-08T23:46:13.000000 Description:Docker volume. Encrypted:false Name:cvol-4 VolumeType:lvmdriver-1 ReplicationDriverData: ReplicationExtendedStatus: ReplicationStatus:disabled SnapshotID: SourceVolID: Status:available TenantID:979ddb6183834b9993954ca6de518c5a Metadata:map[readonly:False] Multiattach:false ID:677a821b-e33b-44b5-8fb3-38315e832216 Size:1 UserID:e763fc47a2854b15a9f6b85aea2ae3f1}\n"
Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=debug msg="Found Volume ID: 677a821b-e33b-44b5-8fb3-38315e832216"
Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=info msg="Remove/Delete Volume: cvol-4"
Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=debug msg="getVolByName: `cvol-4`"
Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=debug msg="querying volume: {Attachments:[] AvailabilityZone:nova Bootable:false ConsistencyGroupID: CreatedAt:2016-09-08T23:46:13.000000 Description:Docker volume. Encrypted:false Name:cvol-4 VolumeType:lvmdriver-1 ReplicationDriverData: ReplicationExtendedStatus: ReplicationStatus:disabled SnapshotID: SourceVolID: Status:available TenantID:979ddb6183834b9993954ca6de518c5a Metadata:map[readonly:False] Multiattach:false ID:677a821b-e33b-44b5-8fb3-38315e832216 Size:1 UserID:e763fc47a2854b15a9f6b85aea2ae3f1}\n"
Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=debug msg="Found Volume ID: 677a821b-e33b-44b5-8fb3-38315e832216"
Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=debug msg="Remove/Delete Volume ID: 677a821b-e33b-44b5-8fb3-38315e832216"
Sep 08 23:46:59 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:46:59Z" level=debug msg="Response from Delete: {ErrResult:{Result:{Body:<nil> Header:map[] Err:<nil>}}}\n"
Sep 08 23:47:07 box-1 cinder-docker-driver[13917]: time="2016-09-08T23:47:07Z" level=info msg="List volumes: "
```



That's about it, the plugin should get picked up by your Docker Daemon and be ready to accept requests.  Couple of things to look out for:

1. You have to use V2.0 as your endpoint for now, GopherCloud chose to implement extensions (like Attach) as versioned items instead of extensions which is what they currently are.   Discovery works as long as it ends up being V2.0, but V1 or V3 won't work.  Microversioning support is going to create some consumer issues here I think but we'll cross that bridge when we come to it.

2. If you get weird error messages on startup about username and/or TenantID it's almost always an issue with your TenantID so start there

3. It never hurts to restart the Docker Daemon on your node after you install the Plugin just to be safe

4. We expect/look for the config.json file in /var/lib/cinder/dockerdriver/ but you can specifiy this with the --config flag.

#### Using the Plugin via Docker

So that's about it as far as install and setup.  Now you can access your Cinder service from the Docker volume API.  This means you can do things like use the Docker Volumes CLI to do things, as well as add Volume arguments to your Docker run commands and even your compose file.

##### Docker Volume CLI

```
$ docker volume --help

Usage:     docker volume COMMAND

Manage Docker volumes

Options:
      --help   Print usage

Commands:
  create      Create a volume
  inspect     Display detailed information on one or more volumes
  ls          List volumes
  rm          Remove one or more volumes

Run 'docker volume COMMAND --help' for more information on a command.
```

Let's look at a simple create example:

```
$ docker volume create -d cinder -o size=1 --name demovol

```


Now list using Dockers API:


```
$ docker volume ls
DRIVER              VOLUME NAME
cinder              demovol
```

Cool.. how about checking with Cinder:

```
$ cinder list

+--------------------------------------+-----------+---------+------+-------------+----------+-------------+
| ID                                   | Status    | Name    | Size | Volume Type | Bootable | Attached to |
+--------------------------------------+-----------+---------+------+-------------+----------+-------------+
| 21a42614-c381-4bbc-b1d0-8498014f2d65 | available | demovol | 1    | lvmdriver-1 | false    |             |
+--------------------------------------+-----------+---------+------+-------------+----------+-------------+
$
```
I know, I know.. big deal, we can create a volume.

Let's try something a little more interesting; let's spin up a container and specify a volume to be created at the same time, we can do this using the Docker Run command, or via a Dockerfile.

##### Docker Run command

Syntax to add a persistent storage volume to your conatiner at launch.  Notice that the Plugin will first check to see if the Volume already exists or not, if it does NOT it will create it for you using the info provided or defaults, if it does exist it will just do the attach and mount for you.

```
$ docker run -v demovol-2:/Data --volume-driver=cinder -i -t ubuntu /bin/bash
$ cinder list

+--------------------------------------+-----------+-----------+------+-------------+----------+--------------------------------------+
| ID                                   | Status    | Name      | Size | Volume Type | Bootable | Attached to                          |
+--------------------------------------+-----------+-----------+------+-------------+----------+--------------------------------------+
| 08c0c2af-4f91-4488-82a7-96bd020e8c00 | in-use    | demovol-2 | 1    | lvmdriver-1 | false    | 969bd5f5-64e3-433e-a248-04c3f7899a45 |
| 21a42614-c381-4bbc-b1d0-8498014f2d65 | available | demovol   | 1    | lvmdriver-1 | false    |                                      |
+--------------------------------------+-----------+-----------+------+-------------+----------+--------------------------------------+
$

```

So now you'll have Cinder volume (demovol-2) mounted and formatted with ext4 at /Data inside your container.

##### docker-compose

I like compose, automation is bestest....

```
web:
  environment:
    - "reschedule:on-node-failure"
    - "constraint:node!=swarm-master"
  image: jgriffith/jgriffith-webbase
  ports:
     - "80:80"
  links:
    - "redis:redis"
redis:
  environment:
    - "reschedule:on-node-failure"
  image: redis
  volumes:
    - "demovol:/data"
  volume_driver: cinder
```

## Conclusion
So these are pretty trivial examples, but it's pretty easy to extend these examples into more *complex* builds.  I've been using this setup for a while now to host a number of things ranging from a [Harbor Docker Registry](https://github.com/vmware/harbor) (images and DB backed by seperate cinder-volumes and running on a Nova Instance).  Other simple things are backing for a Redis store that I use as a simple message queue for a CI system I run, and another application I have where I'm running a Redis DB container that is persisting it's data to a Cinder volume.  There's quite a few things in my lab using Cinder now, some of those things are Nova Instances, but more and more of them lately are something else.

The key take away here however is that Cinder is a lot more flexible than people might think.  It doesn't have any real hard dependencies on Nova, Glance etc. It's just about managing a pool of storage via the control plane and it leaves it up to the consumer to connect it however/wherever it wants.

I'd love to hear your feedback, and try and answer any questions you might have.  Thanks for reading!

## Next time

You can also use this with Swarm, coming up next we'll go through how you can quickly and easily deploy a Swarm cluster on your OpenStack cloud and use the Cinder driver to keep persistent data in your Swarm Cluster and automatically move the volume across nodes in failover scenarios.  In the meantime, if you're interested check it out, feel free to suggest improvements, raise issues or submit PRs.

Before you ask... Kubernetes is very different.  It doesn't actually use Docker's Volume API, and with the exception of some experimental code in the different Cloud Provider drivers it also doesn't do things like dynamic provisioning (create volumes).  It's a completely different animal, and does things it's own way.  TLDR if you're using Kubernetes, this doesn't really buy you anything.

If you want more info on why this, and not some of the existing projects (including Kuryr) I'm happy to provide my opinions...  Thanks!!
