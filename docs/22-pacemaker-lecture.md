---
layout: default
title: Configuring Pacemaker Cluster HA for SAP NetWeaver
nav_order: 22
parent: Day 2
---

# Configuring a Basic High Availability Cluster for SAP Netweaver

<!-- REWORK REQUIRED -->

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

## Setting up HA for SAP NetWeaver

**Create a resource for ASCS instance**

- For ENSA1: When the installation and testing are complete according
  to earlier chapters, you can integrate the SAP NetWeaver system into
  the pacemaker cluster. Assuming that your underlying storage and
  network environment as applicable are configured according to SAP
  guidelines and are part of the cluster, then the following command
  starts the SAP NetWeaver resources into the pacemaker cluster.

<!-- -->

    [root@node ~]# pcs resource create <sid>_ascs<InstanceNumber> SAPInstance \
    > InstanceName="<SID>_ASCS<InstanceNumber>_rhascs" \
    > START_PROFILE=/sapmnt/<SID>/profile/<SID>_ASCS<InstanceNumber>_rhascs \
    > AUTOMATIC_RECOVER=false meta resource-stickiness=5000 migration-threshold=1 \
    > failure-timeout=60 --group <sid>_ASCS<InstanceNumber>_group \
    > op monitor interval=20 on-fail=restart timeout=60 \
    > op start interval=0 timeout=600 \
    > op stop interval=0 timeout=600

The `meta resource-stickiness=5000` value balances out the failover
constraint with ERS, so the resource stays on the node where it started,
and does not migrate around the cluster uncontrollably. The
`migration-threshold=1` value ensures ASCS failover to another node when
an issue is detected instead of restarting on the same node.

- For ENSA2:

      [root@node ~]# pcs resource create <sid>_ascs<InstanceNumber> SAPInstance \
      > InstanceName="<SID>_ASCS<InstanceNumber>_s4ascs" \
      > START_PROFILE=/sapmnt/<SID>/profile/<SID>_ASCS<InstanceNumber>_s4ascs \
      > AUTOMATIC_RECOVER=false \
      > meta resource-stickiness=5000 \
      > --group <sid>_ASCS<InstanceNumber>_group \
      > op monitor interval=20 on-fail=restart timeout=60 \
      > op start interval=0 timeout=600 \
      > op stop interval=0 timeout=600

  Add a resource stickiness value to the group to ensure that the ASCS
  stays on a node if possible:

      [root@node ~]# pcs resource meta <sid>_ASCS<InstanceNumber>_group \
      > resource-stickiness=3000

  Create a resource for an ERS instance
  Create the ERS instance cluster resource.

The `IS_ERS=true` attribute is mandatory for ENSA1 deployments. For more
information about `IS_ERS`, see _How Does the IS_ERS Attribute Work on
an SAP NetWeaver Cluster with Stand-alone Enqueue Server (ENSA1 and
ENSA2)?_: [](https://access.redhat.com/solutions/5474031)

- For ENSA1:

      [root@node ~]# pcs resource create <sid>_ers<InstanceNumber> SAPInstance \
      > InstanceName="<SID>_ERS<InstanceNumber>_rhers" \
      > START_PROFILE=/sapmnt/<SID>/profile/<SID>_ERS<InstanceNumber>_rhers \
      > AUTOMATIC_RECOVER=false IS_ERS=true --group rh2_ERS29_group \
      > op monitor interval=20 on-fail=restart timeout=60 \
      > op start interval=0 timeout=600 \
      > op stop interval=0 timeout=600

- For ENSA2:

      [root@node ~]# pcs resource create s4h_ers29 SAPInstance \
      > InstanceName="S4H_ERS29_s4ers" \
      > START_PROFILE=/sapmnt/S4H/profile/S4H_ERS29_s4ers \
      > AUTOMATIC_RECOVER=false \
      > --group s4h_ERS29_group \
      > op monitor interval=20 on-fail=restart timeout=60 \
      > op start interval=0 timeout=600 \
      > op stop interval=0 timeout=600

**Create the required constraints**

Create a colocation constraint for ASCS and ERS resource groups.

Resource groups `<sid>_ASCS<InstanceNumber>_group` and
`<sid>_ERS<InstanceNumber>_group` should avoid running on the same node
whenever both nodes are available.

    [root@node ~]# pcs constraint colocation add rh2_ERS29_group with \
    > rh2_ASCS20_group -5000

This concludes the chapter on Configuring a Basic High Availability
Cluster for SAP.

## Additional Information

[Configuring and Managing High Availability Clusters](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html-single/configuring_and_managing_high_availability_clusters/index)

[Automating SAP HANA Scale-Up System Replication using the RHEL HA Add-On](https://access.redhat.com/articles/3004101)

[Sysadmin Blog: How to set up a Pacemaker cluster for high availability Linux](https://www.redhat.com/sysadmin/rhel-pacemaker-cluster)
