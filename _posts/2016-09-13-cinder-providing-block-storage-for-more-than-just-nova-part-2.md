---
layout: post
title: Cinder - Block Stroage for things other than Nova (Part-2)
comments: true
---

<meta http-equiv='Content-Type' content='text/html; charset=utf-8' />

In previous posts we started talking about using Cinder as a Block Storage
provider for consumers other than Nova.  In particular our first example has
been how to use Cinder by itself for Docker.  You can catch the first part of
that here [Cinder providing block storage for more than just Nova]({% post_url
2016-09-07-cinder-providing-block-storage-for-more-than-just-nova %})

So there we walked through how to install the [Cinder Docker Driver](https://github.com/j-griffith/cinder-docker-driver/blob/master/README.md) and configure
things in a simple stand-alone setup. In this post we're going to follow up on
a few things and consume Cinder resources from a Swarm Cluster.

If you've never built a Swarm cluster, it's *super* easy using Docker 1.12.
Even better, if you have an OpenStack Cloud internally, you can check out my
post [How to Swarm on OpenStack]({% post_url 2016-09-01-how-to-swarm-on-openstack %}), or watch the [Screen Cast](https://youtu.be/Dh2BK7xm02o) and try
this out for yourself in about 10 minutes or so.

## A little Background

We're going to start from the poing where we already have a Swarm Cluster
set up and ready to use.  For simplicity I'm going to use the Swarm
deployed on my OpenStack Cloud. That means I'm using Nova Instances as
my Swarm Nodes, BUT Nova doesn't know anything about Cinder in this case.

The whole idea here is how to use Cinder just as Cinder by
itself. This all works the same whether you're Swarm is on

 * Bare Metal
 * vSphere
 * VirtualBox
 * OpenStack

**Whoa, hold up!!**

Did I say use Cinder to provide storage for my Containers running in
vSphere???

**YES I DID!!  and YES YOU CAN!**

The whole point of this post is that we only really *need* Cinder.

## Installing/Configuring the cinder-docker-driver on each Docker Node

The following is much the same as we did in part 1 of our post,  we'll go
through the same steps, but we're going to do everything through
docker-machine.  You could certainly do this with your own bash, python or
ansible or *whatever*, but docker-machine and the command line in my case works
really well.

#### Firs we'll use curl to fetch and run the install script for the Plugin

```
$ for each in $(docker-machine ls -q); do; docker-machine ssh $each "curl -sSl https://raw.githubusercontent.com/j-griffith/cinder-docker-driver/master/install.sh | sudo sh -"; done
```

Remember, we're manually creating conf files because of the HostUUID issue.  Since we're using OpenStack Instances
here, might as well use their UUID's for this.  I've created the files I need,
they're all the same with the exception of the ```HostUUID:``` entry:

```
$ for each in 1 2 3 4; do; docker-machine scp cdd.config.swarm-$each.json swarm-$each:~/config.json; done
```

Now, just move the files into the default cinder dockerdriver directory using sudo...

```
$ for each in $(docker-machine ls -q); do; docker-machine ssh $each "sudo cp ~/config.json /var/lib/cinder/dockerdriver/config.json"; done
```

Start the plugin daemon on each Node

```
$ for each in $(docker-machine ls -q); do; docker-machine ssh $each "sudo service cinder-docker-driver start"; done
```

Verify the services all started up ok...

```
$ for each in $(docker-machine ls -q); do; docker-machine ssh $each "sudo service cinder-docker-driver status"; done
● cinder-docker-driver.service - "Cinder Docker Plugin daemon"
   Loaded: loaded (/etc/systemd/system/cinder-docker-driver.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2016-09-13 23:12:52 UTC; 8s ago
 Main PID: 26875 (cinder-docker-d)
    Tasks: 5
   Memory: 1.2M
      CPU: 11ms
   CGroup: /system.slice/cinder-docker-driver.service
           └─26875 /usr/bin/cinder-docker-driver &

Sep 13 23:12:52 swarm-1 systemd[1]: Started "Cinder Docker Plugin daemon".
Sep 13 23:12:52 swarm-1 cinder-docker-driver[26875]: time="2016-09-13T23:12:52Z" level=info msg="Using config file: /var/lib/cinder/dockerdriver/config.json"
Sep 13 23:12:52 swarm-1 cinder-docker-driver[26875]: time="2016-09-13T23:12:52Z" level=info msg="Set InitiatorIFace to: default"
Sep 13 23:12:52 swarm-1 cinder-docker-driver[26875]: time="2016-09-13T23:12:52Z" level=info msg="Set node InitiatorIP to: "
Sep 13 23:12:52 swarm-1 cinder-docker-driver[26875]: time="2016-09-13T23:12:52Z" level=info msg="Set DefaultVolSz to: 1 GiB"
Sep 13 23:12:52 swarm-1 cinder-docker-driver[26875]: time="2016-09-13T23:12:52Z" level=info msg="Set Endpoint to: http://172.16.140.243:5000/v2.0"
Sep 13 23:12:52 swarm-1 cinder-docker-driver[26875]: time="2016-09-13T23:12:52Z" level=info msg="Set Username to: jdg"
Sep 13 23:12:52 swarm-1 cinder-docker-driver[26875]: time="2016-09-13T23:12:52Z" level=info msg="Set TenantID to: 3dce5dd10b414ac1b942aba8ce8558e7"
......
```

Easy enough... we've now succesfully added/configured the Cinder Plugin to each
of our Swarm nodes.  We're ready to deploy Swarm services and consume our
Cinder storage.

## Our first Swarm Service that Consumes Cinder

The first thing we'll do now is set our env back to our Swarm Manager Node, and
create a Swarm network to use.  I just use the basic overlay driver, this
network is really important.  We'll specify this in our services and it will do
things like enable the containers to communicate across nodes, and also do
things like allow us to access a running container from any one of the Swarm
Nodes IP addresses (regardless of where the Container is actually running).


```
$ eval $(docker-machine env swarm-1)
$ docker network create --driver overlay swarm-net
93huy770ajjyvk87hc3kmdabr

$ docker network list
NETWORK ID          NAME                DRIVER              SCOPE
4bc75c8415f5        bridge              bridge              local
a4b3019e592a        docker_gwbridge     bridge              local
4fce4f1ddbe0        host                host                local
0by3ei97kci5        ingress             overlay             swarm
afafb56eb377        none                null                local
93huy770ajjy        swarm-net           overlay             swarm
```

So you'll notice we did a ```docker network list`` there, we want to make sure
we have a network we can use with the SCOPE==swarm.

At this point we're going to build an "app", albeit a very simple app.  Our app
is going to consist of:
1. A redis service (that persists data to a Cinder Volume)
2. A web service that takes input via a simple web-page and stores the info to
   our Redis service

The app is a modified version of the Moby-Counter app you've probably seen.
One thing about Docker is that "counting" apps seem to be THE DEFACTO example,
so why break with tradition here.

### Start our Redis Service

```
docker service create --name redis --network swarm-net --mount
type=volume,src=redis-1,dst=/data,volume-driver=cinder -p 6379:6379 redis
```

So that's pretty simple, the syntax for service as it relates to Volumes is a
little different, but it makes sense once you kinda walk through it.

We want to "mount" something to the container when it starts up
``` --mount ```

That thing we want to mount is a "volume"
``` type=volume ```

```src=redis-1```  Name of the src volume to create, or just use one if
it already exists.

```dst=/data```  That's where we're going to mount the src volume in our
container.

```volume-driver=cinder```  Tells the Docker Volume API *which* Volume
Plugin/Service to use.

That's all ther is to it.  Swarm will figure out where to place the Container
(since we're not giving it any constraints anywhere one of the Swarm nodes is
fair game), And it will communicate with the Cinder Plugin Daemon ON the Node
that it places the Container.

The Plugin takes care of all the details:

1. Check if Volume already exists
2. Create it if it doesn't
3. Perform an iSCSI attach to the Swarm Node
4. Format the volume (only if it's a new Volume that it created)
5. Mount the volume on the Swarm Node
6. Pass the Mount point into the Container.

Now... if that Container "fails" for some reason, it will be restarted on
another node.  In that case the above steps are repeated on the new Node.  So,
you may have lost a node, or your container died, but your storage and
"important" data are now efficiently portable across your Swarm Nodes!!!

### Start up our web server

```
$ docker service create --name web --network swarm-net -p 80:80 \
jgriffith/jgriffith-webbase
```

That's all there is to it, now you should be able to do ```docker service
list``` and ```docker service ps <service-id>``` to get details about where
your service is running and it's status.

## Put it all together and demo in a video

[Screencast demo](http://www.youtube.com/watch?v=XnGfmDQMABs)
