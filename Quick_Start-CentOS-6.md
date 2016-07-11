---
layout: default
title: PostgreSQL Automatic Failover - Quick start CentOS 7
---
# Quick Start CentOS 6

This quick start tutorial is based on CentOS 6.7, using the `pcs` command.


## Network setup

First, we should make sure NetworkManager is disabled. The network setup should
NOT be dynamically handled by some external daemons. Only Pacemaker related
processes should be able to deal with this:

```
chkconfig NetworkManager off
service NetworkManager stop
```

During the cluster setup, we use the cluster name in various places, make sure
all your servers names can be resolved to the correct IPs. We usually set this
in the `/etc/hosts` file:

```
192.168.122.50 pgsql-ha
192.168.122.51 srv1
192.168.122.52 srv2
192.168.122.53 srv3
192.168.123.51 srv1-alt
192.168.123.52 srv2-alt
192.168.123.53 srv3-alt
```

The IP address `192.168.122.50`, called `pgsql-ha` in this tutorial, will be set
on the server hosting the master PostgreSQL intance.

Finally, we have to allow the network traffic related to the cluster and
PostgreSQL to go through the firewalls:

```
service iptables start

# corosync
iptables -I INPUT -p udp -m state --state NEW -m multiport --dports 5404,5405 -j ACCEPT

# pcsd
iptables -I INPUT -p tcp --dport 2224 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -I OUTPUT -p tcp --sport 2224 -m state --state ESTABLISHED -j ACCEPT

# postgres
iptables -I INPUT -p tcp --dport 5432 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -I OUTPUT -p tcp --sport 5432 -m state --state ESTABLISHED -j ACCEPT

service iptables save
```

## PostgreSQL and Cluster stack installation

We are using the PostgreSQL packages from the PGDG repository. Here is how to
install and set up this repository on your system:

```
yum install http://yum.postgresql.org/9.3/redhat/rhel-6-x86_64/pgdg-centos93-9.3-1.noarch.rpm
```

Make sure to double adapt the previous command with the latest package available
and the PostgreSQL version you need.

We can now install everything we need for our cluster:

```
yum install -y corosync pacemaker postgresql93 postgresql93-contrib postgresql93-server resource-agents pcs pcsd cman fence-agents-virsh
```

Finally, we need to install the "PostgreSQL Automatic Failover" (PAF) resource agent:

```
yum install -y https://github.com/dalibo/PAF/releases/download/v1.0.0/resource-agents-paf-1.0.0-1.noarch.rpm
```

## PostgreSQL setup

The resource agent requires the PostgreSQL instances to be already set up and
ready to start. Moreover, it requires a `recovery.conf` template ready to use.
You can create a `recovery.conf` file suitable to your needs, the only
requirements are:

  * have ``standby_mode = on``
  * have ``recovery_target_timeline = 'latest'``
  * a ``primary_conninfo`` with an ``application_name`` set to the node name

Here are some quick steps to build your primary PostgreSQL instance and its
standbys. As this is not the main subject here, they are
__**quick and dirty**__. Rely on the
[PostgreSQL documentation](http://www.postgresql.org/docs/current/static/index.html)
for a proper setup.

On the primary:

```
service postgresql-9.3 initdb

su - postgres

cd 9.3/data/
cat <<EOP >> postgresql.conf

listen_addresses = '*'
wal_level = hot_standby
max_wal_senders = 10
hot_standby = on
hot_standby_feedback = on
EOP

cat <<EOP >> pg_hba.conf
# forbid self-replication
host replication postgres 192.168.122.50/32 reject
host replication postgres $(hostname -s) reject

# allow any standby connection
host replication postgres 0.0.0.0/0 trust
EOP

cat <<EOP > recovery.conf.pcmk
standby_mode = on
primary_conninfo = 'host=192.168.122.50 application_name=$(hostname -s)'
recovery_target_timeline = 'latest'
EOP

exit

service postgresql-9.3 start
ip addr add 192.168.122.50/24 dev eth0
```

Now, on each standby, clone the primary. E.g.:

```
su - postgres

pg_basebackup -h pgsql-ha -D ~postgres/9.3/data/ -X stream -P

cd ~postgres/9.3/data/

sed -ri s/srv[0-9]+/$(hostname -s)/ pg_hba.conf
sed -ri s/srv[0-9]+/$(hostname -s)/ recovery.conf.pcmk

cp recovery.conf.pcmk recovery.conf

exit

service postgresql-9.3 start
```

Finally, make sure to stop the PostgreSQL services __everywhere__ and to
disabling them, as Pacemaker will take care of starting/stopping everything for
you:

```
service postgresql-9.3 stop
chkconfig postgresql-9.3 off
```

And remove the master IP address from `srv1`:

```
ip addr del 192.168.122.50/24 dev eth0
```


## Cluster setup


### Pacemaker

It is advised to keep Pacemaker off on server boot. It helps the administrator
to investigate after a node fencing before Pacemaker starts and potentially
enters in a death match with the other nodes. Make sure to disable Corosync as
well to avoid unexpected behaviors. Run this on all nodes:

```
systemctl disable corosync # important!
systemctl disable pacemaker
```


### Cluster creation

This guide uses the cluster management tool `pcsd` provided by RHEL to ease the
creation and setup of a cluster. It allows to create the cluster from command
line, without editing configuration files or XML by hands.

`pcsd` uses the hacluster system user to work and communicate with other
members of the cluster. We need to set a password to this user so it can
authenticate to other nodes easily. As cluster management commands can be run on
any member of the cluster, it is recommended to set the same password everywhere
to avoid confusions:

```
passwd hacluster
```

Enable and start the `pcsd` daemon on all the nodes:

```
chkconfig pcsd on
service pcsd start
```

Now, authenticate each node to the other ones using the following command:

```
pcs cluster auth srv1 srv2 srv3 -u hacluster
```

We can now create our cluster!

```
pcs cluster setup --name cluster_pgsql srv1 srv2 srv3
```

If you have an alternative network available (this is highly recommended), you
can use the following syntax:

```
pcs cluster setup --name cluster_pgsql srv1,srv1-alt srv2,srv2-alt srv3,srv3-alt
```

If your version of `pcs` does not support it (ie. CentOS 6.6 and bellow), you
can fallback on the old but useful `ccs` command:

```
pcs cluster setup --name cluster_pgsql srv1 srv2 srv3
ccs -f /etc/cluster/cluster.conf --addalt srv1 srv1-alt
ccs -f /etc/cluster/cluster.conf --addalt srv2 srv2-alt
ccs -f /etc/cluster/cluster.conf --addalt srv3 srv3-alt
pcs cluster sync
```

You can now start your cluster!

```
pcs cluster start --all
```

After some seconds of startup and cluster membership stuff, you should be able
to see your three nodes up in `crm_mon` (or `pcs status`):

```
root@srv1:~# crm_mon -n1D
Node srv1: online
Node srv2: online
Node srv3: online
```

We can now feed this cluster with some resources to keep available. This guide
use the cluster client `pcs` to setup everything.


## Cluster resource creation and management

This setup creates three different resources: `pgsql-ha`, `pgsql-master-ip`
and `fence_vm_xxx`.

The `pgsql-ha` resource represents all the PostgreSQL instances of your cluster
and controls where is the primary and where are the standbys.
The `pgsql-master-ip` is located on the node hosting the PostgreSQL master
resource.
The last resources `fence_vm_xxx` are STONITH resources, used to manage fencing.
We create one STONITH resource for each node. Each fencing resource will not be
allowed to run on the node it is supposed to stop. We are using the
`fence_virsh` fencing agent, which is power fencing agent allowing to power on
or off a virtual machine through the `virsh` command. For more information
about fencing, see documentation `docs/FENCING.md` in the source code or
online:
[http://dalibo.github.com/PAF/fencing.html]({{ site.baseurl }}/fencing.html).

First of all, let's create an empty CIB file and fill it with some basic setup.
We will push to the cluster once we are completely done:

```
pcs cluster cib cluster1.xml
pcs -f cluster1.xml resource defaults migration-threshold=5
pcs -f cluster1.xml resource defaults resource-stickiness=10
```

Then, we must start populating it with the STONITH resources (this setup is
only provided as an example, and must be adapted to the fencing agent used in
your configuration):

```
pcs -f cluster1.xml stonith create fence_vm_srv1 fence_virsh pcmk_host_check="static-list" pcmk_host_list="srv1" ipaddr="192.168.122.1" login="root" port="srv1-c6" action="off" identity_file="/root/.ssh/id_rsa"
pcs -f cluster1.xml stonith create fence_vm_srv2 fence_virsh pcmk_host_check="static-list" pcmk_host_list="srv2" ipaddr="192.168.122.1" login="root" port="srv2-c6" action="off" identity_file="/root/.ssh/id_rsa"
pcs -f cluster1.xml stonith create fence_vm_srv3 fence_virsh pcmk_host_check="static-list" pcmk_host_list="srv3" ipaddr="192.168.122.1" login="root" port="srv3-c6" action="off" identity_file="/root/.ssh/id_rsa"
pcs -f cluster1.xml constraint location fence_vm_srv1 avoids srv1=INFINITY
pcs -f cluster1.xml constraint location fence_vm_srv2 avoids srv2=INFINITY
pcs -f cluster1.xml constraint location fence_vm_srv3 avoids srv3=INFINITY
```

We add the PostgreSQL `pgsqld` resource and the multistate `pgsql-ha`
responsible to clone it everywhere and define the roles (master/slave) of each
clone:

```
# pgsqld
pcs -f cluster1.xml resource create pgsqld ocf:heartbeat:pgsqlms \
    bindir=/usr/pgsql-9.3/bin pgdata=/var/lib/pgsql/9.3/data     \
    op start timeout=60s                                         \
    op stop timeout=60s                                          \
    op reload timeout=20s                                        \
    op promote timeout=30s                                       \
    op demote timeout=120s                                       \
    op monitor interval=15s timeout=10s role="Master"            \
    op monitor interval=16s timeout=10s role="Slave"             \
    op notify timeout=60s                                        \
    op meta-data timeout=5s                                      \
    op validate-all timeout=5s                                   

# pgsql-ha
pcs -f cluster1.xml resource master pgsql-ha pgsqld \
    master-max=1 master-node-max=1                  \
    clone-max=3 clone-node-max=1 notify=true
```

Note that the values for `timeout` and `interval` on each operation are based
on the minimum suggested value for PAF Resource Agent.
These values should be adapted depending on the context.

We add the IP address which should be started on the primary node:

```
pcs -f cluster1.xml resource create pgsql-master-ip ocf:heartbeat:IPaddr2 \
    ip=192.168.122.50 cidr_netmask=24 op monitor interval=10s
```

We now define the collocation between `pgsql-ha` and `pgsql-master-ip`.
Note that the start/stop and promote/demote order for these resource is
asymetrical: we __must__ keep the master IP on the master during its demote
process.

```
pcs -f cluster1.xml constraint colocation add pgsql-master-ip with master pgsql-ha INFINITY
pcs -f cluster1.xml constraint order promote pgsql-ha then start pgsql-master-ip symmetrical=false
pcs -f cluster1.xml constraint order demote pgsql-ha then stop pgsql-master-ip symmetrical=false
```

And finally, we define a preference for our master node:

```
pcs -f cluster1.xml constraint location pgsql-ha prefers srv1=1
```

We can now push our CIB to the cluster, which will start all the magic stuff:

```
pcs cluster cib-push cluster1.xml
```

## Adding a node to the cluster

Setup your new node following the first chapters. Stop after the PostgreSQL
setup.

On this new node, setup the pcsd deamon and its authentication:

```
passwd hacluster
chkconfig pcsd on
service pcsd start
pcs cluster auth srv1 srv2 srv3 srv4 -u hacluster
```

On all other nodes, authenticate to the new node:

```
pcs cluster auth srv4 -u hacluster
```

We are now ready to add the new node. Put the cluster in maintenance mode so it
does not move resources all over the place when the new node appears:

```
pcs property set maintenance-mode=true
pcs cluster node add srv4,srv4-alt
```

Or, using the old commands if the syntax of `pcs` with alternate interface is
not supported:

```
pcs cluster node add srv4
ccs -f /etc/cluster/cluster.conf --addalt srv4 srv4-alt
pcs cluster sync
```

And reload the corosync configuration on all the nodes if needed (it just fails
if not needed):

```
pcs cluster reload corosync
```

We now need to allow one more clone in the cluster:

```
pcs resource meta pgsql-ha clone-max=4
```

Add the STONITH agent for the new node:

```
pcs stonith create fence_vm_srv4 fence_virsh pcmk_host_check="static-list" pcmk_host_list="srv4" ipaddr="192.168.122.1" login="root" port="srv4-c6" action="off" identity_file="/root/.ssh/id_rsa"

pcs constraint location fence_vm_srv4 avoids srv4=INFINITY
```

And you can now exit your maintenance mode:

```
pcs property set maintenance-mode=false
```