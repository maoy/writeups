# How to use the ZooKeeper driver for ServiceGroup in OpenStack Nova

## Introduction

If you are reading this page, I assume you have basic knowledge of what OpenStack is. But in case you don't know, OpenStack is a suite of software to turn your data center into a dynamic API driven cloud computing environment. Nova is the project name for OpenStack Compute, one of the key parts of an OpenStack. It manages all your compute nodes, where virtual machines (VMs) are hosted.

## ServiceGroup APIs
To effectively manage and utilize compute nodes, Nova needs to know the status of them. For example, when a user launches a new VM, the Nova scheduler should send the request to a *live* node (with enough capacity too, of course). Starting from the (upcoming) Grizzly release, Nova queries the ServiceGroup API to get the node liveness information.

Here is roughly how the ServiceGroup API works. When a compute worker (running the `nova-compute` daemon) starts, it calls the `join` API to join the compute group, so that whoever is interested in the information (e.g. the scheduler) will be able to query the group membership (by call `get_all` or `get_one`), or the status of a particular node via the `service_is_up` ServiceGroup API call. Under the hood, the ServiceGroup client driver will automatically update the compute worker status. If a certain node should explicitly be removed from the ServiceGroup, it can be done by the `leave` call. 

## ServiceGroup Drivers
Currently there are two drivers implemented and merged in master: database and ZooKeeper. More drivers are either in review (such as the memcached one), or in development. 

### Database ServiceGroup driver

The database driver is how Nova has been tracking node liveness since the first release, and it's the default driver. In a compute worker, it periodically sends a db update command to MySQL, saying "I'm OK" with a timestamp. A pre-defined timeout (`service_down_time`) is used to determine if a node is dead.
The driver has two limitations, which may or may not be an issue for you, depending on your setup. First, the more compute worker nodes you have, the more pressure you put on the database. Second, the timeout is by default 60 seconds. So, it might take a while to detect node failures. Of course you could try to reduce the timeout value, but you would also need to make the db update more frequently, which again increases the db workload.

The fundamental issue is, the data to describe whether the node is alive is "transient" --- it's basically useless after a couple of seconds. Other data in the database, such as the entries to describe which users own what VMs, are persistent. But they are treated the same way since they are stored in the same database. This is also part of the motivation where we started the ServiceGroup abstraction so that we have a chance to treat them separately.

### ZooKeeper ServiceGroup driver

#### How it works
The ZooKeeper ServiceGroup driver works by using ZooKeeper ephemeral nodes. ZooKeeper in cotrary to databases is a distributed system, and its load is divided among several servers.At a compute worker node, after establishing a ZooKeeper sesion, it creates an ephemeral znode in the group directory. Ephemeral znodes has the same lifespan as the session. If the worker node or the `nova-compute` daemon crashes, or there is a network partition between the worker and the ZooKeeper server quorums, then the ephemeral znodes are removed automatically. The driver gets the group membership by doing a "ls" in group directory.

#### Installation and configuration

To use the ZooKeeper driver, of course you need to have both ZooKeeper servers and client libraries installed. Setting up ZooKeeper servers is outside the scope of this article. For the rest of the article, let's assume you have them installed, and their addresses / ports are 192.168.0.1:2181, 192.168.0.2:2181, 192.168.0.3:2181.

To use ZooKeeper, you'll need two client-side Python libraries on every nova node. Here is the command to install then in Ubuntu:
```
sudo apt-get install python-zookeeper python-pip
sudo pip install evzookeeper
```
`python-zookeeper` is the official ZooKeeper Python binding. [evzookeeper](https://github.com/maoy/python-evzookeeper) is the library to make the official binding work with the eventlet threading model.

After installation, make sure you have the following configuration snippet at the end of `/etc/nova/nova.conf` on every node:
```
servicegroup_driver="zk"

[zookeeper]
address="192.168.0.1:2181,192.168.0.2:2181,192.168.0.3:2181"
```

That's it! After you start each Nova components, you can try to use `nova-manage service list` to check the liveness of each compute node.
