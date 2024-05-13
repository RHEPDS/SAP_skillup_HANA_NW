---
layout: default
title: Verifying the Configuration of HA Cluster
nav_order: 30   
parent: Day 4
---

# Verifying the Configuration of HA Cluster Managing SAP Resources

**Outcomes**

After completing this section, you should be able to verify the status
of a basic high availability cluster for managing SAP HANA or SAP
NetWeaver.

All nodes in a Pacemaker cluster must use the same major version of
Red Hat Enterprise Linux. Red Hat Enterprise Linux 8 clusters use
Corosync 3.x for communication. Red Hat Enterprise Linux 7 Pacemaker
clusters use Corosync 2.x.

1.  Run the `pcs status` command to verify the current state of the
    cluster:

        [root@node ~]# pcs status

    To verify the detailed status of the cluster, including the
    configuration, use the `pcs config` command:

        [root@nodea ~]# pcs config

2.  When performing any maintenance activity, you might want to monitor
    the real-time status of the cluster in a separate terminal. Run the
    following command to start the real-time monitoring:

        [root@nodea ~]# crm_mon

    This process occupies the terminal; to return to the prompt, press
    Ctrl+c.

3.  To verify resource constraints, run the `pcs constraint` command:

        [root@nodea ~]# pcs constraint

4.  To review the constraints in more detail, run the
    `pcs constraint --full` command:

        [root@nodea ~]# pcs constraint --full

5.  Move the resource and verify the status:

        [root@nodea ~]# pcs resource move <sap-resource> <Node-Name>

6.  For SAP HANA resources, use the following command for the move:

        [root@nodea ~]# crm_resource --move --resource <sap-hana-resource> \
        > <Node-Name>

7.  To prevent the `myresource` resource group from running on node2,
    execute the following command:

        [root@nodea ~]# pcs resource ban <sap-resource> node2.example.com

    Both `pcs resource move` and `pcs resource ban` create a constraint
    rule on the cluster. Constraints are used, among other reasons, to
    influence which resources can run where. How to list and delete
    constraints is described later in this chapter.

    To clear the ban restriction for the `sapresource` resource group,
    execute the following command:

        [root@nodea ~]# pcs resource clear <sapresource>

    To view current resource defaults, run this command:

        [root@nodea ~]# pcs resource defaults

**Finish**

You have verified the configuration of a RHEL HA cluster for managing
SAP resources.
