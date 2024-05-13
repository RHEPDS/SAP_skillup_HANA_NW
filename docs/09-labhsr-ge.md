---
layout: default
title: Setting up the SAP HSR
nav_order: 9
parent: Day 2
---

# Guided Exercise: Setting up the SAP HSR

In this exercise, you access and use the lab environment, and install
SAP HANA system replication on the previously installed SAP HANA
servers.

**Outcomes**

You write a playbook that installs SAP HANA system replication from the
`hana1` to the `hana2` server.

As the `student` user on the `workstation` machine, use the `lab`
command to prepare your system for this exercise.

This command ensures that the environment is configured correctly for
creating your Ansible Playbooks in the future.

    [student@workstation ~]$ lab start sap-hana-hsr

1.  Change to the `ansible-files` directory in your home directory:

        [student@workstation ~]$ cd ~/ansible-files

2.  Add the following content to the `group_vars/hanas` file:

        ## BEGIN sap_hana_hsr parameters
        sap_hana_cluster_nodes:
          - node_name: hana1.lab.example.com
            node_ip: "172.25.250.22"
            node_role: primary
            hana_site: DC01

          - node_name: hana2.lab.example.com
            node_ip: "172.25.250.23"
            node_role: secondary
            hana_site: DC02
        ## END sap_hana_hsr parameters

    With this configuration, you achieve the following outcome:

    - You define the nodes for system replication. This variable is
      also used later in the cluster setup.

3.  Create the `setup-hsr.yml` playbook to install SAP HANA on the
    servers:

        - name: Step 3-C - Configure HANA System Replication
          hosts: hanas
          become: true

          tasks:

            - name: Configure System Replication
              include_role:
                name: community.sap_install.sap_ha_install_hana_hsr

4.  Execute the `setup-hsr.yml` playbook.

        [student@workstation ~]$ ansible-playbook setup-hsr.yml -v -K

5.  Log in to `hana1` and verify that system replication is working.

        [student@workstation ansible-files]$ ssh hana1
        Last login: Fri Jul 22 04:49:22 2022
        [student@hana1 ~]$ sudo su - rheadm
        [sudo] password for student: student
        Last login: Fr Jul 22 04:49:35 EDT 2022 on pts/0
        rheadm@hana1:/usr/sap/RHE/HDB00> cdpy
        rheadm@hana1:/usr/[...]/python_support> python systemReplicationStatus.py
        |Database |Host  |Port  |Service Name |Volume ID |Site ID |Site Name |Secon   ...
        |         |      |      |             |          |        |          |Host    ...
        |-------- |----- |----- |------------ |--------- |------- |--------- |------  ...
        |SYSTEMDB |hana1 |30001 |nameserver   |        1 |      1 |DC01      |hana2   ...
        |RHE      |hana1 |30007 |xsengine     |        2 |      1 |DC01      |hana2   ...
        |RHE      |hana1 |30003 |indexserver  |        3 |      1 |DC01      |hana2   ...

        status system replication site "2": ACTIVE
        overall system replication status: ACTIVE

        Local System Replication State
        ~~~~~~~~~~

        mode: PRIMARY
        site id: 1
        site name: DC01

    Notice here that data is being replicated from `hana1` to `hana2`,
    and the servers are fully in sync.

6.  Log in to `hana2` and verify the replication state.

        [student@workstation ansible-files]$ ssh hana2
        [student@hana2 ~]$ sudo su - rheadm
        [sudo] password for student: student
        rheadm@hana2:/usr/sap/RHE/HDB00> hdbnsutil -sr_state

        System Replication State
        ~~~~~~~~

        online: true

        mode: sync
        operation mode: logreplay
        site id: 2
        site name: DC02

        is source system: false
        is secondary/consumer system: true
        has secondaries/consumers attached: false
        is a takeover active: false
        is primary suspended: false
        is timetravel enabled: false
        replay mode: auto
        active primary site: 1

        primary masters: hana1

        Host Mappings:
        ~~~~~~

        hana2 -> [DC02] hana2
        hana2 -> [DC01] hana1


        Site Mappings:
        ~~~~~~
        DC01 (primary/primary)
            |---DC02 (sync/logreplay)

        Tier of DC01: 1
        Tier of DC02: 2

        Replication mode of DC01: primary
        Replication mode of DC02: sync

        Operation mode of DC01: primary
        Operation mode of DC02: logreplay

        Mapping: DC01 -> DC02
        done.

    Notice here that `hana2` is the secondary site named `DC02`, and
    `hana1` is the primary server.

7.  Now that both servers are in sync, take over the primary role on
    `hana2`:

        rheadm@hana2:/usr/sap/RHE/HDB00> hdbnsutil -sr_takeover
        done.

    Running this command might take some time.

    After a takeover, both servers are acting as primary servers.
    Therefore, before initiating a takeover, you must ensure that no
    application is writing to `hana1` any more, for example by shutting
    down `hana1` first. In this case, no application exists yet, so it
    is not a concern.

8.  Now you register `hana1` as a new secondary server.

    1.  Log in to `hana1` and assume `rheadm`:

            [student@workstation ansible-files]$ ssh hana1
            [student@hana1 ~]$ sudo su - rheadm
            [sudo] password for student: student

    2.  Stop the HANA database:

            rheadm@hana1:/usr/sap/RHE/HDB00> HDB stop
            hdbdaemon will wait maximal 300 seconds for NewDB services finishing.
            Stopping instance using: /usr/sap/RHE/SYS/exe/hdb/sapcontrol -prot NI_HTTP
             -nr 00 -function Stop 400

            07.09.2022 06:00:42
            Stop
            OK
            Waiting for stopped instance using: /usr/sap/RHE/SYS/exe/hdb/sapcontrol
             -prot NI_HTTP -nr 00 -function WaitforStopped 600 2

            07.09.2022 06:01:12
            WaitforStopped
            OK
            hdbdaemon is stopped.

    3.  Register `hana1` as secondary host to `hana2`:

            rheadm@hana1:/usr/sap/RHE/HDB00>  hdbnsutil -sr_register --name=DC01 \
            > --remoteHost=hana2 --remoteInstance='00' --replicationMode=sync
            --operationMode not set; using default from global.ini/[system_replication]/
            operation_mode: logreplay
            adding site ...
            collecting information ...
            updating local ini files ...
            done.

    4.  Start database again on `hana1`:

            rheadm@hana1:/usr/sap/RHE/HDB00> HDB start


            StartService
            Impromptu CCC initialization by 'rscpCInit'.
              See SAP note 1266393.
            OK
            OK
            Starting instance using: /usr/sap/RHE/SYS/exe/hdb/sapcontrol -prot NI_HTTP -nr 00 -function StartWait 2700 2


            07.09.2022 06:08:13
            Start
            OK

            07.09.2022 06:08:43
            StartWait
            OK

    5.  On `hana1`, verify the replication state:

            rheadm@hana1:/usr/sap/RHE/HDB00> hdbnsutil  -sr_state
            System Replication State
            ~~~~~~~~

            online: true

            mode: sync
            operation mode: logreplay
            site id: 1
            site name: DC01

            is source system: false
            is secondary/consumer system: true
            has secondaries/consumers attached: false
            is a takeover active: false
            is primary suspended: false
            is timetravel enabled: false
            replay mode: auto
            active primary site: 2

            primary masters: hana2

            Host Mappings:
            ~~~~~~

            hana1 -> [DC02] hana2
            hana1 -> [DC01] hana1


            Site Mappings:
            ~~~~~~
            DC02 (primary/primary)
                |---DC01 (sync/logreplay)

            Tier of DC02: 1
            Tier of DC01: 2

            Replication mode of DC02: primary
            Replication mode of DC01: sync

            Operation mode of DC02: primary
            Operation mode of DC01: logreplay

            Mapping: DC02 -> DC01
            done.

9.  On `hana2`, verify the replication state:

        [student@workstation ansible-files]$ ssh hana2
        Last login: Fri Jul 22 04:49:22 2022
        [student@hana2 ~]$ sudo su - rheadm
        [sudo] password for student: student
        Last login: Fr Jul 22 04:49:35 EDT 2022 on pts/0
        rheadm@hana2:/usr/sap/RHE/HDB00> cdpy
        rheadm@hana2:/usr/[...]/python_support> python systemReplicationStatus.py
        |Database |Host  |Port  |Service Name |Volume ID |Site ID |Site Name |Secon   ...
        |         |      |      |             |          |        |          |Host    ...
        |-------- |----- |----- |------------ |--------- |------- |--------- |-----   ...
        |SYSTEMDB |hana2 |30001 |nameserver   |        1 |      2 |DC02      |hana1   ...
        |RHE      |hana2 |30007 |xsengine     |        2 |      2 |DC02      |hana1   ...
        |RHE      |hana2 |30003 |indexserver  |        3 |      2 |DC02      |hana1   ...

        status system replication site "1": ACTIVE
        overall system replication status: ACTIVE

        Local System Replication State
        ~~~~~~~~~~

        mode: PRIMARY
        site id: 2
        site name: DC02

    If the overall system replication status is `ACTIVE`, the system
    replication is configured correctly.

**Finish**

To complete this exercise, take these steps:

- Run the `lab` command on the `workstation` machine, and use the
  `lab` command to create the files in this exercise.

- Run the `ansible-playbook` command to configure HANA system
  replication if not successful previously, and complete the exercise.

These steps are important to ensure that resources from previous
exercises do not impact upcoming exercises.

    [student@workstation ~]$ lab finish sap-hana-hsr
    [student@workstation ~]$ ansible-playbook setup-hsr.yml -v -K
    BECOME password: student
