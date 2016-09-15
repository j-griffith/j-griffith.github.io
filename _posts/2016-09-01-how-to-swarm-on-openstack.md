---
layout: post
title: How to Swarm on OpenStack
comments: true
---

<meta http-equiv='Content-Type' content='text/html; charset=utf-8' />
# Building a SWARM Cluster on your OpenStack Cloud

Also available as a screen-cast with some narrative [here](https://youtu.be/Dh2BK7xm02o)

## Provisioning Instances with docker-machine
Start with an env file (rather than deal with a bunch of cmd line args)

```
export OS_FLAVOR_ID=6
export OS_DOMAIN_NAME=Default
export OS_IMAGE_ID=d5c276bc-cb70-42c4-9291-96f40a03a74c
export OS_SSH_USER=ubuntu
export OS_KEYPAIR_NAME=swarm
export OS_PRIVATE_KEY_FILE=$HOME/.ssh/id_rsa
export OS_SSH_USER=ubuntu
export OS_TENANT_ID=$OS_PROJECT_ID
```

```source dmachine.env```

```
$ source dmachine.env
$ for each in 1 2 3 4; do; docker-machine create -d openstack swarm-$each &; done
Running pre-create checks...
Creating machine...
Running pre-create checks...
Creating machine...
Running pre-create checks...
Creating machine...
Running pre-create checks...
Creating machine...
(swarm-2) Creating machine...
(swarm-1) Creating machine...
(swarm-3) Creating machine...
(swarm-4) Creating machine...

.......

Docker is up and running!
To see how to connect your Docker Client to the Docker Engine running on this
virtual machine, run: docker-machine env swarm-4
```

```
$  docker-machine ls
NAME      ACTIVE   DRIVER      STATE     URL                         SWARM   DOCKER    ERRORS
swarm-1   -        openstack   Running   tcp://172.16.140.107:2376           v1.12.1
swarm-2   -        openstack   Running   tcp://172.16.140.106:2376           v1.12.1
swarm-3   -        openstack   Running   tcp://172.16.140.105:2376           v1.12.1
swarm-4   -        openstack   Running   tcp://172.16.140.108:2376           v1.12.1
```

## Initialize a SWARM Cluster

```
$ eval $(docker-machine env swarm-1)
$ docker swarm init --advertise-addr 172.16.140.107 --listen-addr 172.16.140.107:2377
Swarm initialized: current node (3x8vxeuhnh9mb83hzenudyl70) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
        --token
        SWMTKN-1-51ms3ga9iazxhn8ohh5n6whupxcch1bfdhmkvhnzq5hpzruwup-24hs62n9ih9w0bivvnlyr4k3m
        \
            172.16.140.144:2377

            To add a manager to this swarm, run 'docker swarm join-token
            manager' and follow the instructions.
```
Join each of our nodes as workers

**Repeat the following for each Swarm Node (swarm-<2-4>)**

```
$ eval $(docker-machine env swarm-2)
$ docker swarm join \
    --token
    SWMTKN-1-51ms3ga9iazxhn8ohh5n6whupxcch1bfdhmkvhnzq5hpzruwup-24hs62n9ih9w0bivvnlyr4k3m
    \
        172.16.140.144:2377

        To add a manager to this swarm, run 'docker swarm join-token
        manager' and follow the instructions.
This node joined a swarm as a worker.
```

*THAT'S IT!!!*  You've now got a basic Swarm Cluster depoloyed no your OpenStack Cloud!!
