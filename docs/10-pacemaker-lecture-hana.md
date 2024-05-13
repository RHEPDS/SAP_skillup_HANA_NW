---
layout: default
title: Configuring Pacemaker Cluster HA for SAP HANA Scale Up
nav_order: 10
parent: Day 2
---

# Configuring a High Availability Cluster for SAP HANA Scale Up

In this chapter you will learn the manual steps for an SAP HANA cluster setup on top of the working and running SAP HANA System Replication.

## Install the Node Software

The Red Hat High Availability add-on requires installing the required
set of software packages, configuring the firewall, and authenticating
nodes.

Red Hat Enterprise Linux 8 and Red Hat Enterprise Linux 7 cluster nodes
are not compatible in a single cluster. All nodes in a Pacemaker cluster
must use the same major version of Red Hat Enterprise Linux. Red Hat
Enterprise Linux 8 clusters use Corosync 3.x for communication; Red Hat
Enterprise Linux 7 Pacemaker clusters use Corosync 2.x.

## Install Required Software on the Node to be Part of the Cluster

The `pcs` package provides the cluster configuration software. The `pcs`
package requires the `corosync` and `pacemaker` packages, which are
automatically installed as dependencies for an installation with Yum.
The `fence-agents-all` package pulls in all available fencing agent
packages. Administrators can also choose to install only the
`fence-agents-XYZ` package, where `XYZ` is the intended fencing agent to
use. The `pcs` and `fence-agents-all` packages must be installed on all
the cluster nodes.

    [root@node ~]# yum install pcs fence-agents-all

### Configure the Firewall for Cluster Communication

You can skip this step if you are not using the Linux built-in Firewall.
You must allow cluster communications through any external firewall as
applicable in your environment for all the cluster nodes. The standard
firewall service on a Red Hat Enterprise Linux 8 system is the
`firewalld` service. The `firewalld` daemon ships with a standard
high-availability service for cluster communication. To activate the
`high-availability` firewall service on each of the cluster nodes to
allow cluster communication through the firewall, execute the following
commands:

    [root@node ~]# firewall-cmd --permanent --add-service=high-availability
    [root@node ~]# firewall-cmd --reload

### Enable Pacemaker and Corosync on the Nodes

The `pcsd` service provides the cluster configuration synchronization
and the web front end for cluster configuration. The service is required
on all cluster nodes. Use the `systemctl` command to start and enable
the `pcsd` service on all cluster nodes.

    [root@node ~]# systemctl enable --now pcsd

The `pcsd` service uses the `hacluster` system user for cluster
communication and configuration. You must set the password of the
`hacluster` system user on all cluster nodes. Red Hat recommends to use
the same password for the `hacluster` user on all nodes in the cluster.
The following example sets the `hacluster` user password to `redhat`:

    [root@node ~]# echo redhat | passwd --stdin hacluster

You must authenticate the cluster nodes in the `pcsd` service with the
`hacluster` user and the password that you set up for this user. You
need to run the `pcs host auth` command on only one node to authenticate
all nodes in the cluster.

The `node1.example.com` and `node2.example.com` cluster nodes are
authenticated on the `node1.example.com` system with the `hacluster`
user and the corresponding password.

    [root@node ~]# pcs host auth node1.example.com \
    > node2.example.com
    Username: hacluster
    Password: redhat
    node1.example.com: Authorized
    node2.example.com: Authorized

For automation purposes, the `-u <USERNAME>` and `-p <PASSWORD>` options
can also be used.

## Configure Basic Cluster Communication

After you prepare the two nodes for the cluster setup, the
`pcs cluster setup` command creates the cluster. This command takes as
arguments the cluster name and fully qualified domain names or IP
addresses of the cluster nodes. The optional `--start` parameter starts
the cluster on all supplied cluster nodes.

    [root@node ~]# pcs cluster setup mycluster --start \
    > node1.example.com \
    > node2.example.com

By default, a cluster node that gets rebooted does not automatically
rejoin the cluster. You can use the `pcs cluster enable` command to
enable automatic starting of the cluster service. The `--all` option
enables automatic starting of cluster services on every cluster member.

The following command enables all cluster nodes to start the cluster
service and to automatically join the cluster when executed on one of
the cluster nodes.

    [root@node ~]# pcs cluster enable --all

Red Hat recommends that you verify that the cluster is working as
expected. The `pcs cluster status` command provides an overview of the
current cluster status.

    [root@node ~]# pcs cluster status
    Cluster Status:
     Cluster Summary:
       * Stack: corosync
       * Current DC: node2.example.com (version 2.0.4-6.el8-2deceaa3ae) - partition with quorum
       * Last updated: Fri Mar  5 12:23:08 2021
       * Last change:  Fri Mar  5 12:22:57 2021 by root via cibadmin on node1.example.com
       * 2 nodes configured
       * 0 resource instances configured
     Node List:
       * Online: [ node1.example.com node2.example.com ]

    PCSD Status:
      node1.example.com: Online
      node2.example.com: Online

The `pcs cluster status` command shows the status of all nodes if they
are communicating with each other. The status indicator is the
`Online: [ node1.example.com node2.example.com ]` statement within the
Node List section. Any communication issue between nodes is also
indicated in this section.

## Configure Cluster Node Fencing

Fencing is a requirement for any high availability cluster. It prevents
data corruption from an errant node. Fencing also isolates and restarts
a cluster member if the node fails to join the cluster and the remaining
cluster members still form a quorum. Depending on the hardware used, the
cluster can fence a node by turning off the connection to the shared
storage or by power-cycling the node.

The first step to set up fencing is to configure the physical fencing
device. Different hardware devices are capable of fencing cluster nodes,
for example:

- Uninterruptible power supplies (UPS)
- Power distribution units (PDU)
- Blade power control devices
- Lights-out devices

The fence devices must be added to the cluster. For physical machine
fencing, each cluster node might require its own fence device. Use the
`pcs stonith create` command. The command expects a set of parameter and
value pairs that the fence agent requires to fence the cluster node. To
use the `fence_ipmilan` fencing agent, the `pcmk_host_list`, `username`,
`password`, and `ip` parameters are required. The `pcmk_host_list`
parameter lists the corresponding host as the cluster knows it. The `ip`
parameter expects the IP address or hostname of the fencing device.

For example:

    [root@node ~]# pcs stonith create <fence_device_name> fence_ipmilan \
    > pcmk_host_list=node_private_fqdn \
    > ip=node_IP_BMC \
    > username=username \
    > password=password

The `pcs stonith status` command shows the status of the fence devices
that are attached to the cluster. All `fence_ipmilan` fence devices
should show Started status.

    [root@node ~]# pcs stonith status
      * fence_nodea (stonith:fence_ipmilan):     Started node1.example.com
      * fence_nodeb (stonith:fence_ipmilan):     Started node2.example.com

If the status of any fence device is Stopped, then a communication
problem likely exists between the fencing agent and the fencing server.
Verify the settings of the fence device with the
`pcs stonith config fence_device` command. You can update the settings
with the `pcs stonith update` command.

It is highly recommended to test the fencing even if the devices show
Started state: [](https://access.redhat.com/solutions/18803)

When the testing is complete, you can configure the SAP resources.

Red Hat Enterprise Linux is shipped with many fence devices. You must
verify that your intended fence method is supported for your
environment: [](https://access.redhat.com/articles/2881341)

## Setting up HA for SAP HANA

When the installation and testing are complete, as described in earlier
chapters, the SAP HANA system can be integrated into the pacemaker
cluster. The SAP HANA System Replication needs to be already configured.

Assuming that your underlying storage and network environment as
applicable is configured according to SAP guidelines, the following
command starts the SAP resources into the pacemaker cluster:

    [root@node ~]# pcs resource create SAPHanaTopology_<SID>_<InstanceNumber> \
    > SAPHanaTopology SID=<SID> InstanceNumber=<InstanceNumber> op start \
    > timeout=600 op stop timeout=300 op monitor interval=10 timeout=600 \
    > clone clone-max=2 clone-node-max=1 interleave=true

You can clone a cluster resource to be active on multiple nodes. For
example, you can use cloned resources to configure multiple instances of
an `SAPHanaTopology` resource to distribute throughout a cluster, to
ensure that both nodes have updated information about the SAP HANA
instances. You can clone any resource, provided that the resource agent
supports it. A clone consists of one resource or resource group; in this
case, one resource.

    [root@node ~]# pcs resource create SAPHana_<SID>_<InstanceNumber> SAPHana \
    > SID=<SID> InstanceNumber=<InstanceNumber> PREFER_SITE_TAKEOVER=true \
    > DUPLICATE_PRIMARY_TIMEOUT=7200 AUTOMATED_REGISTER=true op start \
    > timeout=3600 op stop timeout=3600 op monitor interval=61 role="Slave" \
    > timeout=700 op monitor interval=59 role="Master" timeout=700 op promote \
    > timeout=3600 op demote timeout=3600 promotable meta notify=true clone-max=2 \
    > clone-node-max=1 interleave=true

In promotable clone resources, the `promotable` meta attribute is set to
`true`. The instances can then be in one of two operating modes, called
`master` and `slave`. The names of the modes do not have specific
meanings, except that when an instance is started, it must come up in
the `slave` state.

`clone-max`: How many copies of the resource to start. The default is
the number of nodes in the cluster.

`clone-node-max`: How many copies of the resource can start on a single
node. The default value is `1`.

`interleave`: Changes the behavior of ordering constraints (between
clones) so that copies of the first clone can start or stop as soon as
the copy on the same node of the second clone starts or stops (rather
than waiting until every instance of the second clone starts or stops).
Allowed values are `false` or `true`. The default value is `false`.

After successful execution of the previous two commands, the cluster
should look as follows:

    [root@node ~]# pcs status
    .............
      * Clone Set: SAPHanaTopology_<SID>_<InstanceNumber>-clone [SAPHanaTopology_<SID>_<InstanceNumber>]:
        * Started: [ node1.example.com node2.example.com ]
      * Clone Set: SAPHana_<SID>_<InstanceNumber>-clone [SAPHana_<SID>_<InstanceNumber>] (promotable):
        * Masters: [ node1.example.com ]
        * Slaves: [ node2.example.com ]
    .............

The resulting resource should look as follows:

    [root@node ~]# pcs resource config SAPHana_<SID>_<InstanceNumber>-clone
    Clone: SAPHana_<SID>_<InstanceNumber>-clone
     Meta Attrs: clone-max=2 clone-node-max=1 interleave=true notify=true promotable=true
     Resource: SAPHana_<SID>_<InstanceNumber> (class=ocf provider=heartbeat type=SAPHana)
      Attributes: AUTOMATED_REGISTER=true DUPLICATE_PRIMARY_TIMEOUT=180 InstanceNumber=<InstanceNumber> PREFER_SITE_TAKEOVER=true SID=RH2
      Operations: demote interval=0s timeout=3600 (SAPHana_RH2_02-demote-interval-0s)
                  methods interval=0s timeout=5 (SAPHana_RH2_02-methods-interval-0s)
                  monitor interval=61 role=Slave timeout=700 (SAPHana_RH2_02-monitor-interval-61)
                  monitor interval=59 role=Master timeout=700 (SAPHana_RH2_02-monitor-interval-59)
                  promote interval=0s timeout=3600 (SAPHana_RH2_02-promote-interval-0s)
                  start interval=0s timeout=3600 (SAPHana_RH2_02-start-interval-0s)
                  stop interval=0s timeout=3600 (SAPHana_RH2_02-stop-interval-0s)

Refer to the following table for more information about some important
parameters in these commands:

<table>
<colgroup>
<col style="width: 36%" />
<col style="width: 9%" />
<col style="width: 9%" />
<col style="width: 45%" />
</colgroup>
<thead>
<tr class="header">
<th style="text-align: left;">Attribute name</th>
<th style="text-align: left;">Required?</th>
<th style="text-align: left;">Default values</th>
<th style="text-align: left;">Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: left;">SID</td>
<td style="text-align: left;">yes</td>
<td style="text-align: left;">null</td>
<td style="text-align: left;">The SAP System Identifier (SID) of the SAP
HANA installation (must be identical for all nodes). Example: RH2</td>
</tr>
<tr class="even">
<td style="text-align: left;">InstanceNumber</td>
<td style="text-align: left;">yes</td>
<td style="text-align: left;">null</td>
<td style="text-align: left;">The instance number of the SAP HANA
installation (must be identical for all nodes). Example: 02</td>
</tr>
<tr class="odd">
<td style="text-align: left;">PREFER_SITE_TAKEOVER</td>
<td style="text-align: left;">no</td>
<td style="text-align: left;">null</td>
<td style="text-align: left;">Does the resource agent prefer to switch
over to the secondary instance instead of restarting the primary
locally? true: Prefer takeover to the secondary site; false: Prefer
restart locally; never: Under no circumstances do a takeover to the
other node.</td>
</tr>
<tr class="even">
<td style="text-align: left;">AUTOMATED_REGISTER</td>
<td style="text-align: left;">no</td>
<td style="text-align: left;">false</td>
<td style="text-align: left;">If a takeover event occurred, and the
DUPLICATE_PRIMARY_TIMEOUT is expired, register the former primary
instance as secondary? false: No, manual intervention is needed; true:
Yes, the resource agent registers the former primary as secondary.</td>
</tr>
<tr class="odd">
<td style="text-align: left;">DUPLICATE_PRIMARY_TIMEOUT</td>
<td style="text-align: left;">no</td>
<td style="text-align: left;">7200</td>
<td style="text-align: left;">The needed time difference (in seconds)
between two primary time stamps, if a dual-primary situation occurs. If
the time difference is less than the time gap, then the cluster holds
one or both instances in a WAITING status, to give the system
administrator a chance to react to a takeover. After the time difference
elapsed, if AUTOMATED_REGISTER is set to true, then the failed former
primary is registered as secondary. After the registration to the new
primary, the system replication overwrites all data on the former
primary.</td>
</tr>
</tbody>
</table>

### Create Virtual IP Address Resource

A cluster contains a virtual IP address to reach the Master instance of
SAP HANA. Assuming that your internal and external network environment
is properly set up to ensure that the selected IP is reachable from the
client side, you can use the following example command to create a
virtual IPaddr2 resource with a selected IP address of 192.168.0.15:

    [root@node ~]# pcs resource create vip_<SID>_<InstanceNumber> \
    > IPaddr2 ip="192.168.0.15"

The resulting resource should look as follows:

    [root@node ~]# pcs resource config vip_<SID>_<InstanceNumber>
     Resource: vip_<SID>_<InstanceNumber> (class=ocf provider=heartbeat type=IPaddr2)
      Attributes: ip=192.168.0.15
      Operations: start interval=0s timeout=20s (vip_RH2_02-start-interval-0s)
                  stop interval=0s timeout=20s (vip_RH2_02-stop-interval-0s)
                  monitor interval=10s timeout=20s (vip_RH2_02-monitor-interval-10s)

### Create Constraints

**constraint - Start `SAPHanaTopology` before `SAPHana`**

For correct operation, `SAPHanaTopology` resources must be started
before the `SAPHana` resources are started. Also, the virtual IP address
must be present on the node where the Master resource of `SAPHana` is
running. To do so, the following two constraints must be created:

- The `symmetrical=false` attribute defines that only the start of
  resources is of interest, and they do not need to be stopped in
  reverse order.

- Both resources (`SAPHana` and `SAPHanaTopology`) have the
  `interleave=true` attribute that allows parallel starting of these
  resources on nodes. Regardless of ordering, it is not needed to wait
  for all nodes to start `SAPHanaTopology`, and you can start the
  `SAPHana` resource on any nodes as soon as `SAPHanaTopology` is
  running there.

Command for creating the constraint:

    [root@node ~]# pcs constraint order SAPHanaTopology_<SID>_<InstanceNumber>-clone \
    > then SAPHana_<SID>_<InstanceNumber>-clone symmetrical=false

The resulting constraint should look as follows:

    [root@node ~]# pcs constraint
    ...
    Ordering Constraints:
      start SAPHanaTopology_<SID>_<InstanceNumber>-clone then start SAPHana_<SID>_<InstanceNumber>-clone (kind:Mandatory) (non-symmetrical)
    ...

constraint: Colocate the master `IPaddr2` resource with the master `SAPHana` resource  
The following example command colocates the `IPaddr2` resource with the
`SAPHana` resource that was promoted to master.

<!-- -->

    [root@node ~]# pcs constraint colocation add vip_<SID>_<InstanceNumber> \
    > with master SAPHana_<SID>_<InstanceNumber>-clone 2000

The constraint uses a score of 2000 instead of the default INFINITY. It
prevents cluster from taking down the `IPaddr2` resource if no master is
promoted in the `SAPHana` resource. You can still use this address with
tools such as SAP Management Console or SAP Landscape Virtualization
Management that can use this address to query the status information
about the SAP instance.

For more information, see [Colocating Cluster Resources](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html-single/configuring_and_managing_high_availability_clusters/assembly_colocating-cluster-resources.adoc_configuring-and-managing-high-availability-clusters)

The resulting constraint should look as follows:

    [root@node ~]# pcs constraint
    ...
    Colocation Constraints:
      vip_<SID>_<InstanceNumber> with SAPHana_<SID>_<InstanceNumber>-clone (score:2000) (rsc-role:Started) (with-rsc-role:Master)
    ...

### Add a Secondary Virtual IP Address for an Active/Active (Read-Enabled) HANA System Replication (HSR) Setup

Starting with SAP HANA 2.0 SPS1, SAP enables Active/Active
(Read-Enabled) setups for SAP HANA system replication. The secondary
systems of SAP HANA system replication can be used actively for
read-intensive workloads. A second virtual IP address is required to
support such setups, to enable clients to access the secondary SAP HANA
database. To ensure that the secondary replication site can still be
accessed after a takeover, the cluster must move the virtual IP address
around with the slave of the master/slave `SAPHana` resource.

When establishing HSR for the read-enabled secondary configuration, set
the `operationMode` to `logreplay_readaccess`.

**Creating the resource for managing the secondary virtual IP address**

    [root@node ~]# pcs resource create vip2_<SID>_<InstanceNumber> \
    > IPaddr2 ip="192.168.1.11"

Use the appropriate resource agent for managing the IP address based on
the platform where the cluster is running.

- Create location constraints so that the secondary virtual IP address
  is placed on the right cluster node at the right time:

      [root@node ~]# pcs constraint location vip2_<SID>_<InstanceNumber> \
      > rule score=INFINITY hana_<sid>_sync_state eq SOK and hana_<sid>_roles \
      > eq 4:S:master1:master:worker:master
      [root@node ~]# pcs constraint location vip2_<SID>_<InstanceNumber> \
      > rule score=2000 hana_<sid>_sync_state eq PRIM and hana_<sid>_roles eq \
      > 4:P:master1:master:worker:master

- These location constraints ensure that the second virtual IP
  resource has the following behavior:

  - If both a PRIMARY node and a SECONDARY node are available, with
    HANA System Replication as `SOK`, then the second virtual IP
    runs on the SECONDARY node.

  - If the SECONDARY node is not available or the HANA System
    Replication is not `SOK`, then the secondary virtual IP runs on
    the PRIMARY node. If the SECONDARY node is available and the
    HANA System Replication is `SOK` again, then the second virtual
    IP moves back to the SECONDARY node.

  - If the PRIMARY node is not available or the HANA instance that
    runs there has a problem, then after the failover, the SECONDARY
    gets promoted to the PRIMARY role, and the second virtual IP
    continues to run on the same node until the takeover of the
    SECONDARY node is complete and the HANA System Replication is
    `SOK`.

The time is maximized when the second virtual IP resource is assigned to
a node where a healthy SAP HANA instance is running.

This concludes the chapter on configuring an SAP HANA Scale Up cluster.

## Additional Information
[Configuring and Managing High Availability Clusters](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html-single/configuring_and_managing_high_availability_clusters/index)

[Automating SAP HANA Scale-Up System Replication using the RHEL HA Add-On](https://access.redhat.com/articles/3004101)
