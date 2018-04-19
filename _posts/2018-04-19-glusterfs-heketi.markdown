---                                                                                                                                                                                                           
layout:     post
title:      "Quick guide for heketi"
subtitle:   "heketi"
date:       2018-04-19
author:     "Garry fan"
header-img: "img/post-bg-2015.jpg"
tags:
    - glusterfs
    - heketi
---



## Quick guide for heketi

[Heketi] is a project for managing glusterfs cluster.

### Features and pros:

* Written in golang(not sure if this is a pro)
* Provide restful API for cluster/node/device/volume managment
* Support topology configuration
* Support fault tolerate zone configuration
* Support automatic brick arrangement for volume creation

### Cons:

* Single point failure
* Dont support advanced features of glusterfs, such as tiering、nfs sharing
* Aimed for cloud platform backend storage, weak support for NAS

## 1. Build heketi from source 

You can get the binary tar ball from [heketi release].

Here below describes the procedure to build the binary tar ball from source that not mentioned on the officail site, perhaps it's too simple to bring it up there.


	[garry@centos-7]# wget https://dl.google.com/go/go1.10.1.linux-amd64.tar.gz
	[garry@centos-7]# sudo tar -C /usr/local -zxf go1.10.1.linux-amd64.tar.gz
	[garry@centos-7]# echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
	[garry@centos-7]# echo 'export PATH=$PATH:$(go env GOPATH)/bin' >> ~/.bashrc
	[garry@centos-7]# echo 'export GOPATH=$(go env GOPATH)' >> ~/.bashrc
	[garry@centos-7]# source ~/.bashrc
	[garry@centos-7]# mkdir -p $GOPATH/src/github.com/heketi
	[garry@centos-7]# cd $GOPATH/src/github.com/heketi
	[garry@centos-7]# git clone https://github.com/heketi/heketi.git
	[garry@centos-7]# cd heketi
	[garry@centos-7]# sudo yum install glide -y
	[garry@centos-7]# make linux_amd64_dist


When the whole procedure completes without err , 
the target binary tar ball is under the "dist" directory.

## 2. Prepare the glusterfs cluster

In this tutorial we're going to demonstrate how to prepare a glusterfs cluster of 3 nodes

	|-----|-----------|-----------------|----------------|
	| id  | hostname  |   management    |    storage     |
	+:---:+:---------:+:---------------:+:--------------:+
	| 1   | dfs-node1 | 192.168.160.101 | 192.168.24.101 |
	| 2   | dfs-node2 | 192.168.160.102 | 192.168.24.102 |
	| 3   | dfs-node3 | 192.168.160.103 | 192.168.24.103 |
	|-----|-----------|-----------------|----------------|

Install glusterfs-server on each node

	[root@dfs-node1]# yum install -y centos-release-gluster312
	[root@dfs-node1]# yum install -y glusterfs-server
	[root@dfs-node1]# systemctl start glusterd


## 3. Setup heketi

We're going to install heketi on host "dfs-node1"

### Prepare ssh connection with public key

	[root@dfs-node1]# tar zxvf heketi-v6.0.0.linux.amd64.tar.gz -C /opt/
	[root@dfs-node1]# cd /opt/heketi
	[root@dfs-node1]# ssh-keygen -f /opt/heketi/heketi_key -t rsa
	[root@dfs-node1]# ssh-copy-id -i heketi_key.pub root@dfs-node1
	[root@dfs-node1]# ssh-copy-id -i heketi_key.pub root@dfs-node2
	[root@dfs-node1]# ssh-copy-id -i heketi_key.pub root@dfs-node3 

### Heketi configuration

Edit the configuration file "/opt/heketi/heketi.json" according to the following text segments, detailed specification please refer to [Heketi configuration]

	"_jwt": "Private keys for access",
    "jwt": {
      "_admin": "Admin has access to all APIs",
      "admin": {
        "key": "123456"
      },
      "_user": "User only has access to /volumes endpoint",
      "user": {
        "key": "123456"
      }
    },

    "executor": "ssh",

    "_sshexec_comment": "SSH username and private key file information",
    "sshexec": {
      "keyfile": "/opt/heketi/heketi_key",
      "user": "root",
      "port": "22",
      "fstab": "/etc/fstab"
    },

    "db": "/opt/heketi/db/heketi.db",

### Environment variables setup and start heketi

	[root@dfs-node1]# echo "export PATH=$PATH:/opt/heketi" >> ~/.bashrc
    [root@dfs-node1]# source ~/.bashrc
    [root@dfs-node1]# heketi --config=/opt/heketi/heketi.json

### test heketi

	[root@dfs-node1]# curl http://localhost:8080/hello


## 4. Manage glusterfs with heketi utility

After all the preperation, we're here to verify the core functions of heketi

### Configure the topology

It's not our purpose to clarity the notions regarding to topology(you can refer to the official website) but demonstrate the steps of doing it.

First, make a topology configuration file according to the following:

<pre>
{
    "clusters": [
        {
            "nodes": [
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "192.168.160.101"
                            ],
                            "storage": [
                                "192.168.24.101"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/sdb",
                        "/dev/sdc"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "192.168.160.102"
                            ],
                            "storage": [
                                "192.168.24.102"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/sdb",
                        "/dev/sdc"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "192.168.160.103"
                            ],
                            "storage": [
                                "192.168.24.103"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/sdb",
                        "/dev/sdc"
                    ]
                }
           ]
        }
    ]
}
</pre>

Let's make it work:

	[root@dfs-node1]# heketi-cli --server=http://localhost:8080 topology load --json=/opt/heketi/topology.json

Despite all the messages flowing on the console , wait for a few minitues(more or less), finally you'll see it done.
Now it's time to create some volumes.

	[root@dfs-node1]# heketi-cli --server=http://localhost:8080 volume create --size=10

We cant verity the result by typing:

	[root@dfs-node1]# gluster volume info

The gluster tools will show us the detailed information of this freshly created volume

[Heketi]: https://github.com/heketi/heketi
[heketi release]: https://github.com/heketi/heketi/releases
[heketi configuration]: https://github.com/heketi/heketi/blob/master/docs/admin/server.md
