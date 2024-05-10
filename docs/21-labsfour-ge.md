---
layout: default
title: Setting up SAP S/4
nav_order: 0
has_children: true
permalink: /
---

# Guided Exercise: Setting up SAP S/4

In this exercise, you access and use the lab environment, and install
SAP S/4HANA on a NetWeaver server.

**Outcomes**

You write a playbook that installs SAP S/4HANA.

As the `student` user on the `workstation` machine, use the `lab`
command to prepare your system for this exercise.

This command ensures that the environment is configured correctly for
creating your Ansible Playbooks in the future.

    [student@workstation ~]$ lab start sap-s4-install

This process might take some time.

1.  Change to the `ansible-files` directory in your home directory:

        [student@workstation ~]$ cd ~/ansible-files

2.  Add the following content to the `group_vars/s4hanas` file:

        ## BEGIN SAP S/4 install parameters
        # sap_swpm
        #----------
        # Product ID for New Installation
        # This is S4HANA 2021 Foundation
        sap_swpm_product_catalog_id: "NW_ABAP_OneHost:S4HANA2021.FNDN.HDB.ABAP"

        # Do not touch /etc/hosts -> it's already OK
        sap_swpm_update_etchosts: false

        # Software
        sap_swpm_software_path: "/sap-software/S4HANA2021.FNDN"
        sap_swpm_sapcar_path: "{{ sap_swpm_software_path }}"
        sap_swpm_swpm_path: "{{ sap_swpm_software_path }}"

        # NW Passwords
        sap_swpm_master_password: "R3dh4t$123"
        sap_swpm_ddic_000_password: "{{ sap_swpm_master_password }}"
        # HDB Passwords
        sap_swpm_db_system_password: "{{ sap_swpm_master_password }}"
        sap_swpm_db_systemdb_password: "{{ sap_swpm_master_password }}"
        sap_swpm_db_schema_abap_password: "{{ sap_swpm_master_password }}"
        sap_swpm_db_sidadm_password: "{{ sap_swpm_master_password }}"

        # NW Instance Parameters
        sap_swpm_sid: RHE
        sap_swpm_pas_instance_nr: "01"
        sap_swpm_ascs_instance_nr: "02"
        sap_swpm_ascs_instance_hostname: "{{ ansible_hostname }}"
        sap_swpm_fqdn: "{{ ansible_domain }}"

        # HDB Instance Parameters
        # For dual host installation, change the db_host to appropriate value
        sap_swpm_db_host: "hana1"
        sap_swpm_db_sid: RHE
        sap_swpm_db_instance_nr: "00"
        ## END SAP S/4 install parameters

    With this configuration, you achieve the following outcomes:

    - You install S4HANA 2021 Foundation in this exercise, because it
      loads only approximately 16 GB into the HANA database, instead
      of approximately 80 GB for the full ERP system. The Foundation
      has only the basic routines, without application software.

    - You must define the path where the software is located.

    - Set the passwords identically for everything. In production, you
      would use a vault file or Ansible Automation Platform
      credential.

    - Define the SID and the instance numbers of the primary
      application server (PAS) and the central services (ASCS).

    - Note that SAP uses the term "FQDN" for the (fully qualified)
      domain only.

    - Define the connection to the database.

3.  Create the `install-s4.yml` playbook, to install SAP HANA on the
    servers:

        ---
        - name: Install S4
          hosts: nodea.lab.example.com
          become: true

          tasks:
            - name: ensure software mountpoint exists
              file:
                 path: "{{ sap_swpm_software_path }}"
                 state: directory
                 mode: '0755'

            - name: Ensure SAP software directory is mounted
              mount:
                src: "utility:{{ sap_swpm_software_path }}"
                path: "{{ sap_swpm_software_path }}"
                opts: rw
                boot: no
                fstype: nfs
                state: mounted

            - name: execute the SWPM Installation
              include_role:
                name: community.sap_install.sap_swpm

            - name: Ensure SAP software directory is unmounted
              mount:
                path: "{{ sap_swpm_software_path }}"
                state: unmounted

4.  Execute the `install-s4.yml` playbook:

        [student@workstation ansible-files]$ ansible-playbook install-s4.yml -v -K
        BECOME password: student
        ...output omitted...

    The deployment of S/4HANA Foundation might take approximately 30
    minutes. After the installation starts, you see the following
    message:

        TASK [community.sap_install.sap_swpm : SAP SWPM - Wait for sapinst process to exit, poll every 60 seconds] **************************************************
        FAILED - RETRYING: [nodea.lab.example.com]: SAP SWPM - Wait for sapinst process to exit, poll every 60 seconds (1000 retries left).
        FAILED - RETRYING: [nodea.lab.example.com]: SAP SWPM - Wait for sapinst process to exit, poll every 60 seconds (999 retries left).

    These messages occur because the installation is started
    asynchronously, and Ansible probes and waits until the installation
    is successful. It is useful to display the installation log on the
    managed node to during this output.

    To display the installation log run the following commands:

        [student@workstation ansible-files]$ ssh nodea
        [student@nodea ~]$ sudo -i
        [root@nodea ~]# tail -f \
        > $(cat /tmp/sapinst_instdir/.lastInstallationLocation)/sapinst.log

5.  Verify that SAP S/4 is running:

        [student@workstation ~]$ ssh nodea
        [student@nodea ~]$ sudo su - rheadm
        [sudo] password for student: student
        Last login: Di Sep  6 11:38:52 EDT 2022 on pts/0
        nodea:rheadm 5> R3trans -d rheadm
        This is R3trans version 6.26 (release 785 - 25.11.21 - 11:06:00 including rjh702 ).
        unicode enabled version
        R3trans finished (0000).

    If the return value is 0, then everything is fine.

**Finish**

To complete this exercise, take these steps:

- Run the `lab` command on the `workstation` machine, and use the
  `lab` command to create the files in this exercise.

- Run the `ansible-playbook` command to install the S/4HANA server if
  not successful previously, and complete the exercise.

These steps are important to ensure that resources from previous
exercises do not impact upcoming exercises.

    [student@workstation ansible-files]$ lab finish sap-s4-install
    [student@workstation ansible-files]$ ansible-playbook install-s4.yml -v -K
