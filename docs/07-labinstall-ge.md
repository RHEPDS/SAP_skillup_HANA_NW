---
layout: default
title: Setting up SAP HANA
nav_order: 0
has_children: true
permalink: /
---

# Guided Exercise: Setting up SAP HANA

In this exercise, you access and use the lab environment and install SAP
HANA in parallel on both SAP HANA servers.

**Outcomes**

You write a playbook that installs SAP HANA on the `hana1` and `hana2`
servers, which are defined in the `hanas` inventory group.

As the `student` user on the `workstation` machine, use the `lab`
command to prepare your system for this exercise.

This command ensures that the environment is configured correctly for
creating your Ansible playbooks in the future.

    [student@workstation ~]$ lab start sap-hana-install

1.  Change to the `ansible-files` directory in your home directory:

        [student@workstation ~]$ cd ~/ansible-files

2.  Add the following content to the `group_vars/hanas` file:

        ## BEGIN sap_hana_install parameters
        sap_hana_install_software_directory: /sap-software/HANA2SPS06
        sap_hana_install_software_extract_directory: /sap-hana-inst
        sap_hana_install_master_password: "R3dh4t$123"
        sap_hana_sid: "RHE"
        sap_hana_instance_number: "00"
        sap_hana_install_restrict_max_mem: 'y'
        sap_hana_install_max_mem: 38912
        ## END sap_hana_install parameters

    With this configuration, you achieve the following outcomes:

    - The role is looking in the `/software/HANA_installation`
      directory for the installation binaries.

    - The role unpacks the HANA archive to the `/sap-hana-inst`
      directory locally.

    - The database password is set to `R3dh4t$123`.

    - HANA is installed with SID `RHE` and instance number `00`.

    - The memory is limited to 38 GB in this installation.

    Later in the course, you install S/4HANA Foundation on this HANA
    database, which writes only approximately 18 GB into the HANA
    database. The full S/4HANA needs 75 GB, so you would need a 128 GB
    host to work with.

3.  Create the `install-sap-hana.yml` playbook to install SAP HANA on
    the servers:

        - name: Step 3-B - Install SAP HANA
          hosts: hanas
          become: true

          tasks:

            - name: ensure software mountpoint exists
              file:
                 path: "{{ sap_hana_install_software_directory }}"
                 state: directory
                 mode: '0755'

            - name: Ensure SAP software directory is mounted
              mount:
                src: "utility:{{ sap_hana_install_software_directory }}"
                path: "{{ sap_hana_install_software_directory }}"
                opts: ro
                boot: no
                fstype: nfs
                state: mounted

            - name: execute the SAP Hana Installation
              include_role:
                name: redhat.sap_install.sap_hana_install

            - name: Ensure SAP software directory is unmounted
              mount:
                path: "{{ sap_hana_install_software_directory }}"
                state: unmounted

4.  Execute the `install-sap-hana.yml` playbook.

        [student@workstation ansible-files]$ ansible-playbook install-sap-hana.yml -v -K
        BECOME password: student

        ...output omitted...

        PLAY RECAP
        hana1.lab.example.com      : ok=49   changed=7    unreachable=0    failed=0    skipped=86   rescued=0    ignored=0
        hana2.lab.example.com      : ok=49   changed=8    unreachable=0    failed=0    skipped=86   rescued=0    ignored=0

    Be patient. The playbook needs approximately 20 minutes to finish
    installation.

5.  Verify the SAP HANA installation on `hana1` and `hana2`

    1.  Log in to `hana1`

            [student@workstation ansible-files]$ ssh hana1
            [student@hana1 ~]$

    2.  Assume the sidadm role. The SID is RHE, so that the user is
        `rheadm`

            [student@hana1 ~]$ sudo su - rheadm
            [sudo] password for student: student
            rheadm@hana1:/usr/sap/RHE/HDB00>

    3.  Get information on the running HANA database:

            rheadm@hana1:/usr/sap/RHE/HDB00> HDB info
            USER          PID     PPID  %CPU        VSZ        RSS COMMAND
            rheadm      34813    34812   0.0     235320       5308 -sh
            rheadm      35053    34813   0.0     222780       3376  \_ /bin/sh /usr/sap/RHE/HDB00/HDB info
            rheadm      35088    35053   0.0     268532       4064      \_ ps fx -U rheadm -o user:8,pid:8,ppid:8,pcpu:5,vsz:10,rss:10,args
            rheadm      33070        1   0.0     605648      32288 hdbrsutil  --start --port 30003 --volume 3 --volumesuffix mnt00001/hdb00003.00003 --identifier 1662543563
            rheadm      32640        1   0.0     605596      32480 hdbrsutil  --start --port 30001 --volume 1 --volumesuffix mnt00001/hdb00001 --identifier 1662543528
            rheadm      32558        1   0.0      25552       3160 sapstart pf=/hana/shared/RHE/profile/RHE_HDB00_hana1
            rheadm      32565    32558   0.1     509132      82476  \_ /usr/sap/RHE/HDB00/hana1/trace/hdb.sapRHE_HDB00 -d -nw -f /usr/sap/RHE/HDB00/hana1/daemon.ini pf=/usr/sap/RHE/SYS/profile/RHE_HDB00
            rheadm      32587    32565  80.9   12976708    9227456      \_ hdbnameserver
            rheadm      32936    32565   0.5    1229592     161352      \_ hdbcompileserver
            rheadm      32939    32565   158    4845808    4090964      \_ hdbpreprocessor
            rheadm      32975    32565  86.6   13492316    9544808      \_ hdbindexserver -port 30003
            rheadm      32978    32565   2.9    4611388    1333984      \_ hdbxsengine -port 30007
            rheadm      33240    32565   1.4    3424128     480388      \_ hdbwebdispatcher
            rheadm      32478        1   0.6     523588      30932 /usr/sap/RHE/HDB00/exe/sapstartsrv pf=/hana/shared/RHE/profile/RHE_HDB00_hana1 -D -u rheadm
            rheadm@hana1:/usr/sap/RHE/HDB00>

        If these processes are displayed, the installation was
        successful

    4.  Repeat the steps 5.1 - 5.3 on `hana2`

**Finish**

To complete this exercise, take these steps:

- Run the `lab` command on the `workstation` machine, to create the
  files in this exercise.

- Run the `ansible-playbook` command to install the HANA servers if
  not successful previously, and complete the exercise.

These steps are important to ensure that resources from previous
exercises do not impact upcoming exercises.

    [student@workstation ansible-files]$ lab finish sap-hana-install
    [student@workstation ansible-files]$ ansible-playbook install-sap-hana.yml -v -K
    BECOME password: student
