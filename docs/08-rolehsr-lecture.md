---
layout: default
title: Introducing the Concepts and Roles for SAP HANA System Replication
nav_order: 0
has_children: true
permalink: /
---

# Introducing the Concepts and Roles for SAP HANA System Replication

## Overview of Setting up System Replication

SAP HANA system replication is the built-in high availability function.
To enable system replication between two servers, the servers must be
installed identically. You must now name one server as the primary and
the other one as the secondary.

The corresponding role is available on Automation Hub as part of the
`redhat.sap_install` collection v1.1.0+ (supported), or on Ansible
Galaxy as part of the `community.sap_install` collection (upstream,
unsupported).

## Parameters of the SAP HANA System Replication Role

The following parameters are common to the other roles to prepare and
install HANA, and they are also required and reused in this role:

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th style="text-align: left;">Name</th>
<th style="text-align: left;">Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: left;"><code>sap_domain</code></td>
<td style="text-align: left;">The domain name of the SAP systems
(defaults to <code>ansible_domain</code>)</td>
</tr>
<tr class="even">
<td style="text-align: left;"><code>sap_hana_sid</code></td>
<td style="text-align: left;">The SID of the HANA system (such as
<code>RHE</code>)</td>
</tr>
<tr class="odd">
<td style="text-align: left;"><code>sap_hana_instance_number</code></td>
<td style="text-align: left;">The instance number of the HANA system
(such as <code>00</code>)</td>
</tr>
<tr class="even">
<td
style="text-align: left;"><code>sap_hana_install_master_password</code></td>
<td style="text-align: left;">The database system password of both SAP
HANA systems</td>
</tr>
</tbody>
</table>

The following dictionary defines the HANA system replication, and is
also used in the cluster roles to configure the Pacemaker cluster:

     sap_hana_cluster_nodes:
       - node_name: HANA node 1
         node_ip: ÌP of node1 used for system replication
         node_role: [primary|secondary]
         hana_site: name of site

       [...]

The dictionary must contain two or three node definitions, where exactly
one node must have the primary role. All other nodes have the secondary
role.

The default replication mode is `sync`, and uses the `log_replay`
operation mode. You can change this behavior with the following
variables:

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th style="text-align: left;">Name</th>
<th style="text-align: left;">Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td
style="text-align: left;"><code>sap_ha_install_hana_hsr_rep_mode</code></td>
<td style="text-align: left;">Replication mode (defaults to
<code>sync</code>)</td>
</tr>
<tr class="even">
<td
style="text-align: left;"><code>sap_ha_install_hana_hsr_oper_mode</code></td>
<td style="text-align: left;">Operation mode (defaults to
<code>logreplay</code>)</td>
</tr>
</tbody>
</table>

##  

[sap_ha_install_hana_hsr Role
Documentation](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/sap_install/content/role/sap_ha_install_hana_hsr)
