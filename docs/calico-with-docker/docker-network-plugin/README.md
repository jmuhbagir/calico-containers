<!--- master only -->
> ![warning](../../images/warning.png) This document applies to the HEAD of the calico-containers source tree.
>
> View the calico-containers documentation for the latest release [here](https://github.com/projectcalico/calico-containers/blob/v0.14.0/README.md).
<!--- else
> You are viewing the calico-containers documentation for release **release**.
<!--- end of master only -->

# Calico as a Docker network plugin.

This tutorial describes how to set up a Calico cluster in a Docker environment
using Docker's native networking framework (built around libnetwork) with
Calico specific network and IPAM drivers.

The libnetwork networking framework is available in Docker in release 1.9 and
above.

Using the Calico network driver, the required network setup and configuration
for networking containers using Calico is handled automatically as part of the
standard network and container lifecycle.  Provided the network is created 
using the Calico driver, creating a container using that network will 
automatically add the container to the Calico network, creating all necessary
Calico configuration and setting up the interface and routes in the container
accordingly.

The Calico IPAM driver may be used in addition to the the Calico network
driver.  This provides IP address management using the configured Calico IP
Pools as address pools for the container, preferentially selecting sub-blocks
of IPs for a particular host.

## 1. Environment setup

To run through the worked example in this tutorial you will to set up two hosts
with a number of installation dependencies.

Follow the instructions in one of the tutorials below to set up a virtualized
environment using Vagrant or a cloud service - be sure to follow the
appropriate instructions for _Calico as a Docker network plugin_.

- [Vagrant install with CoreOS](../VagrantCoreOS.md)
- [Vagrant install with Ubuntu](../VagrantUbuntu.md)
- [Amazon Web Services (AWS)](../AWS.md)
- [Google Compute Engine (GCE)](../GCE.md)
- [DigitalOcean](../DigitalOcean.md)

Altenatively, you can manually configure your hosts.
- [Manual setup](ManualSetup.md)

If you have everything set up properly you should have `calicoctl` in your 
`$PATH`, and two hosts called `calico-01` and `calico-02`.

## 2. Starting Calico services

Once you have your cluster up and running, start calico on all the nodes, 
specifying the `--libnetwork` option to start the Calico network and IPAM 
driver in a separate container.

On calico-01

    sudo calicoctl node --libnetwork

On calico-02

    sudo calicoctl node --libnetwork

This will start a container on each host. Check they are running

    docker ps

You should see output like this on each node

    vagrant@calico-01:~$ docker ps
    CONTAINER ID        IMAGE                           COMMAND                  CREATED             STATUS              PORTS               NAMES
    eec9ebbfb486        calico/node-libnetwork:latest   "./start.sh"             21 seconds ago      Up 19 seconds                           calico-libnetwork
    ffe6cb403e9b        calico/node:latest              "/sbin/my_init"          21 seconds ago      Up 20 seconds                           calico-node

## 3. Create the networks

This worked example creates three Docker networks, where containers on a 
particular network are isolated from containers in the other networks.

### 3.1 Select the IPAM driver

Before we create the networks, we need to select which IP address management
(IPAM) driver we will use.  There are two options, where we suggest using the
Calico IPAM driver where possible.

#### The Calico IPAM driver

The Calico IPAM driver provides address assignment that is geared towards
a Calico deployment where scaling is important, and port-mapping for the
containers is not required.

The Calico IPAM driver assigns addresses with host aggregation - this is a more
efficient approach for Calico requiring fewer programmed routes.  IPv6
addresses are supported, although due limitations with Docker, it is not 
possible to have an IPv6-only network, and (unlike the IPv4 behavior) it is
necessary to specify a subnet from which to assign addresses.

When running in a cloud environment we need to also set `ipip` and 
`nat-outgoing` options. If using the Calico IPAM driver `calico`, the
`ipip` and `nat-outgoing` options are configured on the Calico IP Pool.

#### The default IPAM driver.

Using the default IPAM driver will allow containers on the network to use
Docker's port mapping feature.  However, in some circumstances packets
routed between containers on different networks on the same host may not go 
via the host vRouter and therefore not have Calico policy applied.

When running in a cloud environment we need to also set `ipip` and 
`nat-outgoing` options. If using the default IPAM driver, `ipip` and 
`nat-outgoing` are specified as options on the `network create`.

### 3.2 Create the network

So, with the IPAM driver selected, we can start creating some networks.

We specify the Calico networking driver (`calico`) when creating the network,
and optionally specify the Calico IPAM driver (`calico`) if you chose to use
Calico IPAM.  

For this worked example, we explicitly choose an CIDR for each network 
rather than using default selections - this is to avoid potential conflicts
with the default NAT IP assignment used by VirtualBox.  Depending on your
specific environment, you may need to choose different CIDRs.

So, once you have decided which type of network to create, following the
appropriate instructions for one of *a)*, *b)*, *c)* or *d)*.

#### a. Networking using Calico IPAM in a non-cloud environment

For Calico IPAM in a non-cloud environment, you need to first create a Calico
IP Pool with no additional options.  Here we create a pool with CIDR
192.168.0.0/16.

    calicoctl pool add 192.168.0.0/16
    
To create the networks, run:

    docker network create --driver calico --ipam-driver calico net1
    docker network create --driver calico --ipam-driver calico net2
    docker network create --driver calico --ipam-driver calico net3
    
#### b. Networking using Calico IPAM in a cloud environment

For Calico IPAM in a cloud environment (AWS, DigitalOcean, GCE), you need to 
first create a Calico IP Pool using the `calicoctl pool add` command specifying
the `ipip` and `nat-outgoing` options.  Here we create a pool with CIDR
192.168.0.0/16.

    calicoctl pool add 192.168.0.0/16 --ipip --nat-outgoing

To create the networks, run:

    docker network create --driver calico --ipam-driver calico net1
    docker network create --driver calico --ipam-driver calico net2
    docker network create --driver calico --ipam-driver calico net3

#### c. Networking using default IPAM in a non-cloud environment

For default IPAM in a non-cloud environment, run: 

    docker network create --driver calico --subnet=192.168.0.0/24 net1
    docker network create --driver calico --subnet=192.168.1.0/24 net2
    docker network create --driver calico --subnet=192.168.2.0/24 net3
    
#### d. Networking using default IPAM in a cloud environment

For default IPAM in a cloud environment (AWS, DigitalOcean, GCE), run:

    docker network create --driver calico --opt nat-outgoing=true --opt ipip=true --subnet=192.168.0.0/24 net1
    docker network create --driver calico --opt nat-outgoing=true --opt ipip=true --subnet=192.168.1.0/24 net2
    docker network create --driver calico --opt nat-outgoing=true --opt ipip=true --subnet=192.168.2.0/24 net3


## 4. Create the workloads in the networks

With the networks created, let's start some containers on each host spread
between these networks.

On calico-01

    docker run --net net1 --name workload-A -tid busybox
    docker run --net net2 --name workload-B -tid busybox
    docker run --net net1 --name workload-C -tid busybox

On calico-02

    docker run --net net3 --name workload-D -tid busybox
    docker run --net net1 --name workload-E -tid busybox

By default, networks are configured so that their members can communicate with 
one another, but workloads in other networks cannot reach them.  A, C and E are
all in the same network so should be able to ping each other.  B and D are in 
their own networks so shouldn't be able to ping anyone else.

## 5. Validation
    
On calico-01 check that A can ping C and E.  We can ping workloads within a 
containers networks by name.

    docker exec workload-A ping -c 4 workload-C.net1
    docker exec workload-A ping -c 4 workload-E.net1

Also check that A cannot ping B or D.  This is slightly trickier because the
hostnames for different networks will not be added to the host configuration of
the container - so we need to determine the IP addresses assigned to containers
B and D.

Since A and B are on the same host, we can run a single command that inspects 
the IP address and issues the ping.  On calico-01

    docker exec workload-A ping -c 4  `docker inspect --format "{{ .NetworkSettings.Networks.net2.IPAddress }}" workload-B`
    
These pings will fail.

To test connectivity between A and D which are on different hosts, it is 
necessary to run the `docker inspect` command on the host for D (calico-02) 
and then run the ping command on the host for A (calico-01).
    
On calico-02

    docker inspect --format "{{ .NetworkSettings.Networks.net3.IPAddress }}" workload-D
    
This returns the IP address of workload-D.

On calico-01

    docker exec workload-A ping -c 4 <IP address of D>

replacing the `<...>` with the appropriate IP address of D.  These pings will
fail.

To see the list of networks, use

    docker network ls
        

## IPv6 (Optional)

IPv6 networking is also supported.  If you are using IPv6 address spaces as
well, start your Calico node passing in both the IPv4 and IPv6 addresses of
the host.

For example:

    calicoctl node --ip=172.17.8.101 --ip6=fd80:24e2:f998:72d7::1 --libnetwork 

See the [IPv6 tutorial](IPv6.md) for a worked example.


## Advanced network policy

If you are using both the Calico network driver and the Calico IPAM driver
it is possible to apply advanced policy to the network.

For more details, read  
[Accessing Calico policy with Calico as a network plugin](AdvancedPolicy.md).


## Further reading

For details on configuring Calico for different network topologies and to
learn more about Calico under-the-covers please refer to the 
[Further Reading](../../../README.md#further-reading) section on the main
documentation README.

[![Analytics](https://ga-beacon.appspot.com/UA-52125893-3/calico-containers/docs/calico-with-docker/docker-network-plugin/README.md?pixel)](https://github.com/igrigorik/ga-beacon)