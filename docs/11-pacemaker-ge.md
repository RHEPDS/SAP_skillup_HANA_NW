---
layout: default
title: Deploying and Configuring the Pacemaker Cluster
nav_order: 11
parent: Day 2
---

# Guided Exercise: Deploying and Configuring the Pacemaker Cluster

## Outcomes

After completing this section, you should be able to install and start a
basic high availability cluster for managing SAP HANA.

As the `student` user on the `workstation` machine, use the `lab`
command to prepare your system for this exercise.

This command ensures that the environment is configured correctly for
creating your Ansible Playbooks in the future.

```bash
[student@workstation ~]$ lab start ansible-pacemaker
```

You now configure a High Availability cluster to manage previously
created HANA resources by using the latest cluster HA roles.

Red Hat Enterprise Linux 8 and Red Hat Enterprise Linux 7 cluster nodes
are not compatible in a single cluster. All nodes in a Pacemaker cluster
must use the same major version of Red Hat Enterprise Linux. Red Hat
Enterprise Linux 8 clusters use Corosync 3.x for communication, although
Red Hat Enterprise Linux 7 Pacemaker clusters use Corosync 2.x.

### Ensure all aliases for the cluster are distributed to all nodes

For running SAP HANA in a cluster we need to define virtual ip adresses for these services in /etc/hosts.

1. Verify that the ansible collection community.sap_install is installed in version 1.4.0

    ```bash
    [student@workstation ~]$ ansible-galaxy collection list
    Collection               Version
    ------------------------ -------
    ansible.posix            1.4.0
    community.general        5.5.0
    redhat.rhel_system_roles 1.22.0
    redhat.sap_install       1.1.0

    # /home/student/.ansible/collections/ansible_collections
    Collection                Version
    ------------------------- -------
    community.sap_install     1.4.0
    community.sap_launchpad   1.0.0
    community.sap_libs        1.1.0
    community.sap_operations  0.9.0
    ```

2. Change to the `ansible-files` directory in your home directory:

    ```bash
    [student@workstation ~]$ cd ~/ansible-files
    ```

3. Create the file `group_vars/all` with the following content

    {% raw % }

    ```yaml
    # Virtual IP addresses
    sap_service_vips:
      - node_ip: 172.25.250.80
        node_name: hana
        state: present
        alias_mode: overwrite
      - node_ip: 172.25.250.81
        node_name: s4ascs
        state: present
        alias_mode: overwrite
      - node_ip: 172.25.250.82
        node_name: s4ers
        state: present
        alias_mode: overwrite
    ```

    {% endraw%}

4. Create a playbook `update_host_aliases.yml` with the following content:

    {% raw %}

    ```yaml
    - name: Update host aliases
      hosts: localhost,all
      become: true

      tasks:
        - name: Update Hostaliases
          ansible.builtin.include_role:
            name: community.sap_install.sap_maintain_etc_hosts
          vars:
            sap_maintain_etc_hosts_list: "{{ sap_service_vips }}"
    ```

    {% endraw %}

    NOTE: you may need to install ansible.utils collection

5. Execute the playbook:

    ```bash
    [student@workstation ansible-files]$ ansible-playbook update_host_aliases.yml -v -K
    BECOME password: student
    ```

### Install the pacemaker cluster on the HANA servers

1. Change to the `ansible-files` directory:

    ```bash
    [student@workstation ~]$ cd ~/ansible-files
    ```

2. Update the `group_vars/hanas` file with the following variables:

    {% raw %}

    ```yaml
    ## BEGIN pacemaker parameters
    sap_ha_pacemaker_cluster_system_roles_collection: 'redhat.rhel_system_roles'
    ha_cluster_cluster_name: cluster1
    ha_cluster_hacluster_password: 'my_hacluster'

    sap_ha_pacemaker_cluster_vip_hana_primary_ip_address: "172.25.250.80"

    sap_ha_pacemaker_cluster_stonith_custom:
      - name: "fence_hana1"
        agent: "stonith:fence_ipmilan"
        options:
          ip: bmc-hana1
          pcmk_host_list: hana1
          power_timeout: 180
          username: admin
          password: password
          lanplus: 1
      - name: "fence_hana2"
        agent: "stonith:fence_ipmilan"
        options:
          ip: bmc-hana2
          pcmk_host_list: hana2
          power_timeout: 180
          username: admin
          password: password
          lanplus: 1
    ## END pacemaker parameters
    ```

    {% endraw %}

3. Create the playbook `setup-pacemaker.yml`:

    ```bash
    [student@workstation roles]$ cd ~/ansible-files
    [student@workstation ansible-files]$ vim setup-pacemaker.yml
    ```

    Add this content

    {% raw %}

    ```yaml
    ---
    - name: "03-D HA Cluster deployment on a 2-node cluster"
      hosts: hanas
      become: true

      tasks:
          - name: Execute Cluster Setup Role
            ansible.builtin.include_role:
              name: community.sap_install.sap_ha_pacemaker_cluster
            vars:
              sap_ha_pacemaker_cluster_system_roles_collection: redhat.rhel_system_roles
              sap_ha_pacemaker_cluster_vip_client_interface: eth0
    ```

    {% endraw %}

4. Execute the playbook:

    ```bash
    [student@workstation ansible-files]$ ansible-playbook setup-pacemaker.yml -v -K
    BECOME password: student
    ```

5. After the successful completion of the playbook, verify the cluster
    state.

    1. Log in to the `hana1` node as the root user with `redhat` as the
       password:

            [student@workstation ~]$ ssh root@hana1

    2. Verify the cluster status with the following command:

        ```bash
        [root@hana1 ~]# pcs status --full
        Cluster name: cluster1
        Cluster Summary:
          * Stack: corosync
          * Current DC: hana2.lab.example.com (2) (version 2.0.5-9.el8-ba59be7122) - partition with quorum
          * Last updated: Thu May 11 05:35:35 2023
          * Last change:  Thu May 11 05:35:06 2023 by root via crm_attribute on hana1.lab.example.com
          * 2 nodes configured
          * 7 resource instances configured

        Node List:
          * Online: [ hana1.lab.example.com (1) hana2.lab.example.com (2) ]

        Full List of Resources:
          * res_fence_hana1 (stonith:fence_ipmilan):     Started hana1.lab.example.com
          * res_fence_hana2 (stonith:fence_ipmilan):     Started hana2.lab.example.com
          * vip_RHE_00_primary  (ocf::heartbeat:IPaddr2):    Started hana1.lab.example.com
          * Clone Set: SAPHanaTopology_RHE_00-clone [SAPHanaTopology_RHE_00] (promotable):
            * SAPHanaTopology_RHE_00    (ocf::heartbeat:SAPHanaTopology):    Slave hana2.lab.example.com
            * SAPHanaTopology_RHE_00    (ocf::heartbeat:SAPHanaTopology):    Slave hana1.lab.example.com
          * Clone Set: SAPHana_RHE_00-clone [SAPHana_RHE_00] (promotable):
            * SAPHana_RHE_00    (ocf::heartbeat:SAPHana):    Slave hana2.lab.example.com
            * SAPHana_RHE_00    (ocf::heartbeat:SAPHana):    Master hana1.lab.example.com

        Node Attributes:
          * Node: hana1.lab.example.com (1):
            * hana_rhe_clone_state              : PROMOTED
            * hana_rhe_op_mode                  : logreplay
            * hana_rhe_remoteHost               : hana2
            * hana_rhe_roles                    : 4:P:master1:master:worker:master
            * hana_rhe_site                     : DC01
            * hana_rhe_srmode                   : sync
            * hana_rhe_sync_state               : PRIM
            * hana_rhe_version                  : 2.00.067.01.1682405377
            * hana_rhe_vhost                    : hana1
            * lpa_rhe_lpt                       : 1683797706
            * master-SAPHana_RHE_00             : 150
          * Node: hana2.lab.example.com (2):
            * hana_rhe_clone_state              : DEMOTED
            * hana_rhe_op_mode                  : logreplay
            * hana_rhe_remoteHost               : hana1
            * hana_rhe_roles                    : 4:S:master1:master:worker:master
            * hana_rhe_site                     : DC02
            * hana_rhe_srmode                   : sync
            * hana_rhe_sync_state               : SOK
            * hana_rhe_version                  : 2.00.067.01.1682405377
            * hana_rhe_vhost                    : hana2
            * lpa_rhe_lpt                       : 30
            * master-SAPHana_RHE_00             : 100

        Migration Summary:

        Tickets:

        PCSD Status:
          hana1.lab.example.com: Online
          hana2.lab.example.com: Online

        Daemon Status:
          corosync: active/enabled
          pacemaker: active/enabled
          pcsd: active/enabled
        ```

## Finish

You have configured a basic High Availability cluster for SAP HANA.
