# Installing Swift #

You have been provided with a lab environment that consists of a chef
server, a swift management node, a swift proxy, and three swift
storage nodes.

First, log on to the chef server as root using the provided password.

Take a look around.

    knife node list
	knife cookbook list
	knife role list
    knife environment list
    knife environment edit swift-private-cloud

We've already uploaded a minimal environment and the cookbooks you
will need.  We've taken the liberty of installing chef client on all
of your nodes as well.

You can proceed directly to the fun part -- installing swift.


## The initial runs ##

We'll start with the management server.

First, look at the role

    knife role show spc-starter-controller

The controller is somewhat special.  It is used as a syslog
consolidation server as well as a mail forwarder in the config.  As a
result, its recipes are not very composable with proxies or storage
node recipes, that are configured as clients of the controller.  This
is one area where composability is somewhat weak.

You can see that the keystone recipe is included.  For environments
that will use an existing keystone (from a compute cluster, for
example), you can remove this role and provide some extra attributes.
We will not be doing this in any of the labs, but the process is
documented in environments.md.

Add the role to your controller's run_list:

    knife node run_list add labN-swift-management-server role[spc-starter-controller]

Log on to the controller and run chef-client.  Wait for the run to
complete.  Chat amongst yourselves.  Discuss your amazing learnings.
Poke around at some of the other .md files in this repo.  Tell Will
Kelly how much you love this training so far.  But not sarcastically.
That would hurt his feelings.

Next, the storage nodes.

    for i in labN-swift-storage-server-{1..3}; do
	    knife node run_list add $i role[spc-starter-storage]
    done

Now for the proxy node.  You can have as many of these as you like,
but the cookbooks do not provide a load balancing solution.  It is
expected that the customer will provide a loadbalanced vip.  Since you
get to set the endpoints up in the environment, you can easily
configure the endpoints to use a vip instead of an individual proxy
server.  We aren't doing that here, though.

    knife node run_list add labN-swift-proxy-servers-1 role[spc-starter-proxy]

Log on to all of these nodes and run chef-client.

Wait for all of the chef-clients to finish and then a good 30 seconds
or so for solr indexing to catch up on the chef server, then run
chef-client once more on the admin node.  The admin node has dsh /
pssh configured for the swiftops user to ease your cluster
administration needs.  The final chef run on the admin node updates
these configs.


## Create the ring ##

Even though our disks are not yet configured, we know what the ring is
going to look like for these servers, so let's make it.  The support
wiki has information on some provided scripts for use in production
deployments to make this process easier.  There are some important
decisions that go into picking some of these numbers.  This is well
documented in the swift documentation as well as on the support wiki
and Marcelo will likely go into this in his training.

Here, though, we are going to take advantage of the fact that this is
a throwaway deployment and worry about nothing at all.

Please note that you will need the 192.168 address for each of the storage nodes.

    export STORAGE1=192.168.122.X
	export STORAGE2=192.168.122.Y
	export STORAGE3=192.168.122.Z
	cd /etc/swift

    swift-ring-builder object.builder create 8 3 0
    swift-ring-builder container.builder create 8 3 0
    swift-ring-builder account.builder create 8 3 0

    swift-ring-builder object.builder add z1-${STORAGE1}:6000/disk1 100
    swift-ring-builder object.builder add z2-$(ip_for_host storage2):6000/disk1 100
    swift-ring-builder object.builder add z3-${STORAGE3}:6000/disk1 100

    swift-ring-builder container.builder add z1-${STORAGE1}:6001/disk1 100
    swift-ring-builder container.builder add z2-${STORAGE2}:6001/disk1 100
    swift-ring-builder container.builder add z3-${STORAGE3}:6001/disk1 100

    swift-ring-builder account.builder add z1-${STORAGE1}:6002/disk1 100
    swift-ring-builder account.builder add z2-${STORAGE2}:6002/disk1 100
    swift-ring-builder account.builder add z3-${STORAGE3}:6002/disk1 100

    swift-ring-builder object.builder rebalance
    swift-ring-builder container.builder rebalance
    swift-ring-builder account.builder rebalance

    chown -R swift: .

## Configure your disks! ##

Your storage nodes all have unpartitioned space on /dev/xvda.
Partition these drives--use the remainder of the space in a new
partition!  Run the partprobe.  Format them with the xfs!  On real
deployments, you will want to be careful to use -i size=512 on your
mkfs.xfs, but here it doesn't really matter.  We provide some helpful
scripts for formatting disks as part of the cookbooks.  See the
support wiki or ask Marcelo for more information!

Once you are done:

    mkdir -p /srv/node/disk1
    mount /dev/xvda2 /srv/node/disk1 -o noatime,nodiratime,nobarrier,logbufs=8
	chown -R swift: /srv/node/disk1


## Copy the rings over! ##

On all of the storage nodes and the proxy server, run:

    /usr/bin/swift-ring-minion-server -f -o

Now that the rings have been copied over, swift services will actually
start appropriately.

Run chef-client once more on all of the storage and proxy nodes to get
the services started.


## Profit ##

At this point, you should have a working swift installation.

On the admin server, we drop an openrc file in /root.  Source it and
play with the swift cli tool a little.  Enjoy your victory.  Proceed
to the next lab, in which you can play with the environment!  Or skip
that one and jump directly to the middleware lab, which will actually
provide you with some interesting swift commands to run.
