---
layout: default
title: Setting up SAP S/4 distributed HA
nav_order: 22
parent: Day 3
---

# Guided Exercise: Setting up SAP S/4 distributed HA

In this exercise, you access and use the lab environment, and install
a distributed SAP S/4HANA with RHEL HA (pacemaker).

## Outcomes

You write a playbooks that installs SAP S/4HANA with distributed HA.

As the `student` user on the `workstation` machine, use the `lab`
command to prepare your system for this exercise.

This command ensures that the environment is configured correctly for
creating your Ansible Playbooks in the future.

    [student@workstation ~]$ lab start sap-s4-install

This process might take some time.
It desinstalls S/4 HANA single node installation from previous step

### Update the Ansible inventory

1. Update your inventory to create dedicated groups for ASCS, ERS, PAS and AAS

        [student@workstation ~]$ sudo vi /etc/ansible/hosts

   Modify your inventory that it looks like this:

    ```yaml
    # ansible hosts file for RH455

    [hanas]
    hana1.lab.example.com
    hana2.lab.example.com

    [s4ascs]
    nodea.lab.example.com

    [s4ers]
    nodeb.lab.example.com

    [s4pas]
    nodec.lab.example.com

    [s4aas]
    noded.lab.example.com

    [s4hanas:children]
    s4ascs
    s4ers
    s4pas
    s4aas
    ```

2. Check that the groups contain the right servers:

    ```bash
    [student@workstation ansible-files]$ ansible all --list-hosts
      hosts (4):
        hana1.lab.example.com
        hana2.lab.example.com
        nodea.lab.example.com
        nodeb.lab.example.com
        nodec.lab.example.com
        noded.lab.example.com
    [student@workstation ansible-files]$ ansible s4hanas --list-hosts
      hosts (2):
        nodea.lab.example.com
        nodeb.lab.example.com
    [student@workstation ansible-files]$ ansible s4ascs --list-hosts
      hosts (1):
        nodea.lab.example.com
    [student@workstation ansible-files]$ ansible s4ers --list-hosts
      hosts (1):
        nodeb.lab.example.com
    [student@workstation ansible-files]$ ansible s4pas --list-hosts
      hosts (1):
        nodec.lab.example.com
    [student@workstation ansible-files]$ ansible s4aas --list-hosts
      hosts (1):
        noded.lab.example.com
    ```

NOTE: This is one variant, another would be to create this dynamically inside the playbook.

For running SAP Netweaver in a cluster we need to define virtual ip adresses for ASCS and ERS services in /etc/hosts. We already have created those in the HANA cluster session.

The installation of the netweaver cluster is much more complex to setup than a HANA cluster. It has seven phases:

1. In an ASCS/ERS cluster we need to define the Filesystems and VIPs that will be managed by the cluster before the installation ephemerally. Mount the shared filesystems on all S/4 HANA servers. In our lab we use a prepared NFS Server to serve these directories.
2. Install ASCS on nodea with the VIP s4ascs
3. Install ERS on nodeb with the VIP S4ers
4. Install the pacemaker cluster on top
5. Load the Database on nodec (PAS)
6. Install PAS on nodec
7. (optional) install AAS on noded

Although we could write a single playbook for all these tasks, we do it step by step for a better understanding.
You can find more sophisticated playbooks on the [project source page](https://github.com/sap-linuxlab/ansible.playbooks_for_sap).

### Phase 1: Enable VIPs, ephemral and common mountpoints

1. Change to the `ansible-files` directory in your home directory:

        [student@workstation ~]$ cd ~/ansible-files

2. Create or extend the file `group_vars/s4hanas` with the following content

    {% raw %}
    ```yaml
    ### If the hostname setup is not configured correctly
    #   you need set sap_ip and sap_domain.
    #   we use the full qualified domain name in the inventory
    #   so we can generate these variables
    sap_hostname: "{{ inventory_hostname.split('.')[0] }}"
    sap_domain: "{{ inventory_hostname.split('.')[1:]| join('.') }}"

    ### redhat.sap_install.sap_general_preconfigure
    sap_general_preconfigure_modify_etc_hosts: true
    sap_general_preconfigure_update: true
    sap_general_preconfigure_fail_if_reboot_required: false
    sap_general_preconfigure_reboot_ok: true

    ### redhat.sap_install.sap_netweaver_preconfigure
    sap_netweaver_preconfigure_fail_if_not_enough_swap_space_configured: false

    ## Info for S4HANA HA
    # Software Installation Path
    sap_swpm_software_path: "/sap-software/S4HANA2021.FNDN"
    sap_swpm_sapcar_path: "{{ sap_swpm_software_path }}"
    sap_swpm_swpm_path: "{{ sap_swpm_software_path }}"

    # NFS Server
    sap_storage_nfs_server: '172.25.250.220'
    sap_nwas_shared_mount: '172.25.250.220:/sap-software/mounts'

    ####
    # Mandatory parameters : Virtual instance names
    ####

    # See virtual hostname information in SAP Note 2279110 and 962955
    # Ensure this does not contain the local hostname, must use the Virtual Hostname for use with the Virtual IP (VIP)
    sap_swpm_ascs_instance_hostname: "s4ascs"
    sap_swpm_ers_instance_hostname: "s4ers"
    sap_swpm_db_host: "hana"

    # However we are not using the PAS or AAS in a High Availability setup, only the Database and ASCS/ERS are.
    # Therefore, override the virtual hostname with the local hostname.
    sap_swpm_pas_instance_hostname: "nodec"
    sap_swpm_aas_instance_hostname: "noded"

    ####
    # Mandatory parameters : Virtual IPs (VIPs)
    ####
    sap_ha_pacemaker_cluster_vip_hana_primary_ip_address: "172.25.250.80"
    sap_ha_pacemaker_cluster_vip_nwas_abap_ascs_ip_address: "172.25.250.81"
    sap_ha_pacemaker_cluster_vip_nwas_abap_ers_ip_address: "172.25.250.82"

    # The servers are mutlihomed => Interface for VIP needs to be defined
    sap_ha_pacemaker_cluster_vip_client_interface: eth0

    ####
    # Mandatory parameters : SAP SWPM installation using Default Templates mode of the Ansible Role
    ####

    sap_swpm_ansible_role_mode: default_templates

    # Override any variable set in sap_swpm_inifile_dictionary
    # NW Passwords
    sap_swpm_master_password: "R3dh4t$123"
    sap_swpm_ddic_000_password: "{{ sap_swpm_master_password }}"

    # HDB Configuration
    sap_swpm_db_schema_abap: "SAPHANADB"

    # HDB instance
    sap_swpm_db_sid: "RHE"
    sap_swpm_db_instance_nr: "00"

    # HDB Passwords
    sap_swpm_db_schema_abap_password: "{{ sap_swpm_master_password }}"
    sap_swpm_db_sidadm_password: "{{ sap_swpm_master_password }}"
    sap_swpm_db_system_password: "{{ sap_swpm_master_password }}"
    sap_swpm_db_systemdb_password: "{{ sap_swpm_master_password }}"

    # Unix User ID (optional check from HANA installation)
    sap_swpm_sapadm_uid: '3000'
    sap_swpm_sapsys_gid: '3001'
    sap_swpm_sidadm_uid: '3001'

    # Other
    sap_swpm_fqdn: "{{ ansible_domain }}"
    sap_swpm_update_etchosts: 'false'
    sap_swpm_sid: RHE
    sap_swpm_ascs_instance_nr: "01"
    sap_swpm_ers_instance_nr: "02"
    sap_swpm_pas_instance_nr: "10"
    sap_swpm_aas_instance_nr: "20"

    ####
    # Mandatory parameters : Ansible Dictionary for SAP SWPM installation using Default Templates mode of the Ansible Role
    ####

    # Templates and default values
    sap_swpm_templates_install_dictionary:
      sap_s4hana_2021_distributed_nwas_ascs_ha:
        # Product ID for New Installation
        sap_swpm_product_catalog_id: NW_ABAP_ASCS:S4HANA2021.FNDN.HDB.ABAPHA

        sap_swpm_inifile_list:
          - swpm_installation_media
          - swpm_installation_media_swpm2_hana
          - credentials
          - credentials_hana
          - db_config_hana
          - db_connection_nw_hana
          - nw_config_other
          - nw_config_central_services_abap
          - nw_config_primary_application_server_instance
          - nw_config_ports
          - nw_config_host_agent
          - sap_os_linux_user

        sap_swpm_inifile_dictionary:

          # NW Instance Parameters
          # sap_swpm_sid: "{{ sap_swpm_sid }}"
          # sap_swpm_virtual_hostname: "{{ sap_swpm_ascs_instance_hostname }}"
          # sap_swpm_ascs_instance_nr: "01"
          # sap_swpm_pas_instance_nr: "10"

          # HDB Instance Parameters
          # sap_swpm_db_sid: "RHE"
          # sap_swpm_db_instance_nr: "00"

          # SAP Host Agent
          sap_swpm_install_saphostagent: 'true'

        software_files_wildcard_list:
          - 'SAPCAR*'
          - 'IMDB_CLIENT*'
          - 'SWPM20*'
          - 'igsexe_*'
          - 'igshelper_*'
          - 'SAPEXE_*' # Kernel Part I (785)
          - 'SAPEXEDB_*' # Kernel Part I (785)
          - 'SUM*'
          - 'SAPHOSTAGENT*'


      sap_s4hana_2021_distributed_nwas_ers_ha:
        # Product ID for New Installation
        sap_swpm_product_catalog_id: NW_ERS:S4HANA2021.FNDN.HDB.ABAPHA

        sap_swpm_inifile_list:
          - swpm_installation_media
          - credentials
          - nw_config_other
          - nw_config_ers
          - sap_os_linux_user

        sap_swpm_inifile_dictionary:

          # NW Instance Parameters
          #sap_swpm_sid: "RHE"
          #sap_swpm_virtual_hostname: "{{ sap_swpm_ers_instance_hostname  }}"
          #sap_swpm_ascs_instance_nr: "01"
          #sap_swpm_pas_instance_nr: "10"
          #sap_swpm_ers_instance_hostname: "{{ sap_swpm_ers_instance_hostname }}"
          #sap_swpm_ers_instance_nr: "02"

          # HDB Instance Parameters
          #sap_swpm_db_sid: "RHE"
          #sap_swpm_db_instance_nr: "00"

          # SAP Host Agent
          sap_swpm_install_saphostagent: 'true'

        software_files_wildcard_list:
          - 'SAPCAR*'
          - 'IMDB_CLIENT*'
          - 'SWPM20*'
          - 'igsexe_*'
          - 'igshelper_*'
          - 'SAPEXE_*' # Kernel Part I (785)
          - 'SAPEXEDB_*' # Kernel Part I (785)
          - 'SUM*'
          - 'SAPHOSTAGENT*'


      sap_s4hana_2021_distributed_nwas_pas_dbload_ha:
        # Product ID for New Installation
        sap_swpm_product_catalog_id: NW_ABAP_DB:S4HANA2021.FNDN.HDB.ABAPHA

        sap_swpm_inifile_list:
          - swpm_installation_media
          - swpm_installation_media_swpm2_hana
          - credentials
          - credentials_hana
          - db_config_hana
          - db_connection_nw_hana
          - nw_config_other
          - nw_config_central_services_abap
          - nw_config_primary_application_server_instance
          - nw_config_ports
          - nw_config_host_agent
          - sap_os_linux_user

        sap_swpm_inifile_dictionary:
          # NW Instance Parameters
          # sap_swpm_sid: "RHE"
          # sap_swpm_virtual_hostname: "{{ ansible_hostname }}"
          # sap_swpm_ascs_instance_nr: "01"
          # sap_swpm_pas_instance_nr: "10"

          # HDB Instance Parameters
          # sap_swpm_db_sid: "RHE"
          # sap_swpm_db_instance_nr: "00"

          # SAP Host Agent
          sap_swpm_install_saphostagent: 'true'

        software_files_wildcard_list:
          - 'SAPCAR*'
          - 'IMDB_CLIENT*'
          - 'SWPM20*'
          - 'igsexe_*'
          - 'igshelper_*'
          - 'SAPEXE_*' # Kernel Part I (785)
          - 'SAPEXEDB_*' # Kernel Part I (785)
          - 'SUM*'
          - 'SAPHOSTAGENT*'
          - 'S4CORE*'
          - 'S4HANAOP*'
          - 'HANAUMML*'
          - 'K-*'
          - 'KD*'
          - 'KE*'
          - 'KIT*'
          - 'SAPPAAPL*'
          - 'SAP_BASIS*'


      sap_s4hana_2021_distributed_nwas_pas:
        # Product ID for New Installation
        sap_swpm_product_catalog_id: NW_ABAP_CI:S4HANA2021.FNDN.HDB.ABAP

        sap_swpm_inifile_list:
          - swpm_installation_media
          - swpm_installation_media_swpm2_hana
          - credentials
          - credentials_hana
          - db_config_hana
          - db_connection_nw_hana
          - nw_config_other
          - nw_config_central_services_abap
          - nw_config_primary_application_server_instance
          - nw_config_ports
          - nw_config_host_agent
          - sap_os_linux_user

        sap_swpm_inifile_dictionary:

          # NW Instance Parameters
          # sap_swpm_sid: "RHE"
          # sap_swpm_virtual_hostname: "{{ ansible_hostname }}"
          # sap_swpm_ascs_instance_nr: "01"
          # sap_swpm_pas_instance_nr: "10"

          # HDB Instance Parameters
          # sap_swpm_db_sid: "RHE"
          # sap_swpm_db_instance_nr: "00"

          # SAP Host Agent
          sap_swpm_install_saphostagent: 'true'

        software_files_wildcard_list:
          - 'SAPCAR*'
          - 'IMDB_CLIENT*'
          - 'SWPM20*'
          - 'igsexe_*'
          - 'igshelper_*'
          - 'SAPEXE_*' # Kernel Part I (785)
          - 'SAPEXEDB_*' # Kernel Part I (785)
          - 'SUM*'
          - 'SAPHOSTAGENT*'


      sap_s4hana_2021_distributed_nwas_aas:

        # Product ID for New Installation
        sap_swpm_product_catalog_id: NW_DI:S4HANA2021.FNDN.HDB.PD

        sap_swpm_inifile_list:
          - swpm_installation_media
          - swpm_installation_media_swpm2_hana
          - credentials
          - credentials_hana
          - db_config_hana
          - db_connection_nw_hana
          - nw_config_ports
          - nw_config_other
          - nw_config_additional_application_server_instance
          - nw_config_host_agent
          - sap_os_linux_user

        sap_swpm_inifile_dictionary:

          # NW Instance Parameters
          # sap_swpm_sid: "RHE"
          # sap_swpm_virtual_hostname: "{{ ansible_hostname }}"
          # sap_swpm_ascs_instance_nr: "01"
          # sap_swpm_pas_instance_nr: "10"

          # HDB Instance Parameters
          # sap_swpm_db_sid: "RHE"
          # sap_swpm_db_instance_nr: "00"

          # Product ID suffix is not .ABAP, therefore set variables manually for sap_swpm Ansible Role to populate inifile.params
          sap_swpm_db_schema: "{{ sap_swpm_db_schema_abap }}"
          sap_swpm_db_schema_password: "{{ sap_swpm_db_schema_abap_password }}"

          # SAP Host Agent
          sap_swpm_install_saphostagent: 'true'

        software_files_wildcard_list:
          - 'SAPCAR*'
          - 'IMDB_CLIENT*'
          - 'SWPM20*'
          - 'igsexe_*'
          - 'igshelper_*'
          - 'SAPEXE_*' # Kernel Part I (785)
          - 'SAPEXEDB_*' # Kernel Part I (785)
          - 'SUM*'
          - 'SAPHOSTAGENT*'


    ### pacemaker specific fencing settings
    sap_ha_pacemaker_cluster_stonith_custom:
      - name: "fence_nodea"
        agent: "stonith:fence_ipmilan"
        options:
          ip: bmc-nodea
          pcmk_host_list: nodea
          power_timeout: 180
          username: admin
          password: password
          lanplus: 1
      - name: "fence_nodeb"
        agent: "stonith:fence_ipmilan"
        options:
          ip: bmc-nodeb
          pcmk_host_list: nodeb
          power_timeout: 180
          username: admin
          password: password
          lanplus: 1
    ```
    {% endraw %}

    With this configuration, you achieve the following outcomes:

    - You install S4HANA 2021 Foundation in this exercise, because it loads only approximately 16 GB into the HANA database, instead of approximately 80 GB for the full ERP system. The Foundation has only the basic routines, without application software.

    - You must define the path where the software is located.

    - Set the passwords identically for everything. In production, you would use a vault file or Ansible Automation Platform credential.

    - Define the SID and the instance numbers of the primary and additional
      application server (PAS/AAS) and the central services (ASCS/ERS).

    - Note that SAP uses the term "FQDN" for the (fully qualified) domain only.

    - Define the connection to the database.

    - Define the different ProductIDs and configurations for the different services

    - Define the VIPs, cluster configurations and fencing

3. Create a playbook `install-s4-ha-phase1.yml` with the following content:

  ```yaml
  ---
  # Ansible Playbook for SAP S/4HANA Distributed HA installation

  # Use include_role / include_tasks inside Ansible Task block, instead of using roles declaration or Task block with import_roles.
  # This ensures Ansible Roles, and the tasks within, will be parsed in sequence instead of parsing at Playbook initialisation.

  ## Temporary IP and FS setup for ASCS and ERS
  - name: Add temporary IP to ASCS and mount directories
    hosts: s4ascs
    become: true
    any_errors_fatal: true
    max_fail_percentage: 0
    vars:
      ip_cidr_prefix: 24
    tasks:
      - name: Add ASCS service ip temporary
        command: 'ip address add {{ sap_ha_pacemaker_cluster_vip_nwas_abap_ascs_ip_address }}/{{ ip_cidr_prefix }} brd + dev {{ sap_ha_pacemaker_cluster_vip_client_interface }}'
        register: __ipstate
        changed_when: __ipstate.rc == 0
        failed_when: __ipstate.rc != 0 and __ipstate.rc != 2

      - name: Ensure ASCS directory exists
        ansible.builtin.file:
          path: "/usr/sap/{{ sap_swpm_sid }}/ASCS{{ sap_swpm_ascs_instance_nr }}"
          state: directory
          owner: "{{ sap_swpm_sidadm_uid | d('root') }}"
          group: "{{ sap_swpm_sapsys_gid | d('root') }}"
          mode: '0775'

      - name: Mount ASCS filesystem via nfs
        ansible.posix.mount:
          path: "/usr/sap/{{ sap_swpm_sid }}/ASCS{{ sap_swpm_ascs_instance_nr }}"
          src: "{{ sap_nwas_shared_mount }}/usr/sap/{{ sap_swpm_sid }}/ASCS{{ sap_swpm_ascs_instance_nr }}"
          fstype: nfs
          state: mounted

      # Only necessary if mount state ephemeral not available
      - name: Remove fstab entry for ASCS mount
        ansible.builtin.lineinfile:
          path: /etc/fstab
          state: absent
          regexp: '^{{ sap_nwas_shared_mount }}/usr/sap/{{ sap_swpm_sid }}/ASCS{{ sap_swpm_ascs_instance_nr }} .*$'

  - name: Add temporary IP to ERS and mount directories
    hosts: s4ers
    become: true
    any_errors_fatal: true
    max_fail_percentage: 0
    vars:
      ip_cidr_prefix: 24

    tasks:
      - name: Add ERS service ip temporary
        command: 'ip address add {{ sap_ha_pacemaker_cluster_vip_nwas_abap_ers_ip_address }}/{{ ip_cidr_prefix }} brd + dev {{ sap_ha_pacemaker_cluster_vip_client_interface }}'
        register: __ipstate
        changed_when: __ipstate.rc == 0
        failed_when: __ipstate.rc != 0 and __ipstate.rc != 2

      - name: Ensure ERS directory exists
        ansible.builtin.file:
          path: "/usr/sap/{{ sap_swpm_sid }}/ERS{{ sap_swpm_ers_instance_nr }}"
          state: directory
          owner: "{{ sap_swpm_sidadm_uid | d('root') }}"
          group: "{{ sap_swpm_sapsys_gid | d('root') }}"
          mode: '0775'

      - name: Mount ERS filesystem via nfs
        ansible.posix.mount:
          path: "/usr/sap/{{ sap_swpm_sid }}/ERS{{ sap_swpm_ers_instance_nr }}"
          src: "{{ sap_nwas_shared_mount }}/usr/sap/{{ sap_swpm_sid }}/ERS{{ sap_swpm_ers_instance_nr }}"
          fstype: nfs
          state: mounted

      - name: Remove fstab entry for ERS mount
        ansible.builtin.lineinfile:
          path: /etc/fstab
          state: absent
          regexp: '^{{ sap_nwas_shared_mount }}/usr/sap/{{ sap_swpm_sid }}/ERS{{ sap_swpm_ers_instance_nr }} .*$'

  #### VM storage filesystem setup ####
  - name: Hosts shared storage setup
    hosts: s4hanas
    become: true
    any_errors_fatal: true
    max_fail_percentage: 0
    vars:
      shared_paths:
        - /usr/sap/{{ sap_swpm_sid }}/SYS
        - /usr/sap/trans
        - /sapmnt
    tasks:
      - name: Ensure mountpoints for shared paths exist
        # Content suggestion provided by Ansible Lightspeed
        ansible.builtin.file:
          path: "{{ item }}"
          state: directory
          owner: "{{ sap_swpm_sidadm_uid | d('root') }}"
          group: "{{ sap_swpm_sapsys_gid | d('root') }}"
          mode: '0775'
        loop: "{{ shared_paths |flatten(levels=1) }}"

      - name: Ensure filesystems for shared paths are mounted
        # Content suggestion provided by Ansible Lightspeed
        ansible.posix.mount:
          path: "{{ item }}"
          src: "{{ sap_nwas_shared_mount + item }}"
          fstype: nfs
          state: mounted
        loop: "{{ shared_paths |flatten(levels=1) }}"

      - name: Install rsync
        ansible.builtin.package:
          name:
            - rsync
          state: present
    ```

4.  Execute the `install-s4-ha-phase1.yml` playbook:

        [student@workstation ansible-files]$ ansible-playbook install-s4.yml -v -K
        BECOME password: student
        ...output omitted...

    NOTE: After the swpm installation starts, you see the following
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

5. Create and execute the playbook install-s4-ha-phase2-ascs.yml

  ```yaml
  - name: Ansible Play for SAP NetWeaver Application Server installation - ABAP Central Services (ASCS) for HA
    hosts: s4ascs
    become: true
    any_errors_fatal: true
    max_fail_percentage: 0
    tasks:
      - name: ensure software mountpoint exists
        ansible.builtin.file:
          path: "{{ sap_swpm_software_path }}"
          state: directory
          mode: '0755'

      - name: Ensure SAP software directory is mounted
        ansible.posix.mount:
          src: "utility:{{ sap_swpm_software_path }}"
          path: "{{ sap_swpm_software_path }}"
          opts: rw
          boot: no
          fstype: nfs
          state: mounted

      - name: Execute Ansible Role sap_swpm
        ansible.builtin.include_role:
          name: community.sap_install.sap_swpm
        vars:
          sap_swpm_templates_product_input: "sap_s4hana_2021_distributed_nwas_ascs_ha"
          sap_swpm_virtual_hostname: "{{ sap_swpm_ascs_instance_hostname }}"

      - name: Ensure SAP software directory is unmounted
        mount:
          path: "{{ sap_swpm_software_path }}"
          state: unmounted
    ```

    ```bash
    [student@workstation ansible-files]$ ansible-playbook -v -K install-s4-ha-phase2*
    BECOME password: student
    ... output omitted ...
    ```

7. Create and execute the playbook install-s4-ha-phase3-ers.yml
    ```yaml
    - name: Ansible Play for SAP NetWeaver Application Server installation - ABAP Central Services (ASCS) for HA
      hosts: s4ers
      become: true
      any_errors_fatal: true
      max_fail_percentage: 0
      tasks:
        - name: ensure software mountpoint exists
          ansible.builtin.file:
            path: "{{ sap_swpm_software_path }}"
            state: directory
            mode: '0755'

        - name: Ensure SAP software directory is mounted
          ansible.posix.mount:
            src: "utility:{{ sap_swpm_software_path }}"
            path: "{{ sap_swpm_software_path }}"
            opts: rw
            boot: no
            fstype: nfs
            state: mounted

        - name: Execute Ansible Role sap_swpm
          ansible.builtin.include_role:
            name: community.sap_install.sap_swpm
          vars:
            sap_swpm_templates_product_input: "sap_s4hana_2021_distributed_nwas_ers_ha"
            sap_swpm_virtual_hostname: "{{ sap_swpm_ers_instance_hostname }}"

        - name: Ensure SAP software directory is unmounted
          mount:
            path: "{{ sap_swpm_software_path }}"
            state: unmounted
      ```

      ```bash
      [student@workstation ansible-files]$ ansible-playbook -v -K install-s4-ha-phase3*
      BECOME password: student
      ... output omitted ...
      ```

8. Create and execute the playbook install-s4-ha-phase4-cluster.yml

    ```yaml
    - name: Configure High Availability using ABAP Central Services (ASCS) and Enqueue Replication Service (ERS) with Standalone Enqueue Server 2 (ENSA2)
      hosts: s4ers,s4ascs
      become: true
      any_errors_fatal: true
      max_fail_percentage: 0
      tasks:
        # Execute setup of SAP NetWeaver ASCS/ERS HA cluster
        # -- Linux Pacemaker cluster preparation
        # -- Linux Pacemaker basic cluster configuration, 2 nodes
        # -- SAP NetWeaver ASCS/ERS HA configuration, with a Virtual IP (VIP)
        # -- Fencing Agent/s setup for Infrastructure: fence_aws
        # -- Resource Agent/s setup for Infrastructure: aws-vpc-move-ip
        # -- Resource Agent/s setup for SAP: Filesystem, SAPInstance
        # -- SAP NetWeaver ASCS host:
        # ----> Filesystem for /sapmnt
        # ----> Filesystem for /usr/sap/trans
        # ----> Filesystem for /usr/sap/{{ sap_system_id }}/SYS
        # ----> Filesystem for /sapmnt
        # ----> Filesystem for /usr/sap/{{ sap_system_id }}/ASCS{{ sap_system_nwas_abap_ascs_instance_nr }}
        # ----> SAPInstance for /sapmnt/{{ sap_system_sid }}/profile/{{ sap_system_sid }}_ASCS{{ sap_system_nwas_abap_ascs_instance_nr }}
        # -- SAP NetWeaver ERS host:
        # ----> Filesystem for /usr/sap/{{ sap_system_id }}/ERS{{ sap_system_nwas_abap_ers_instance_nr }}
        # ----> SAPInstance for /sapmnt/{{ sap_system_sid }}/profile/{{ sap_system_sid }}_ERS{{ sap_system_nwas_abap_ers_instance_nr }}

        - name: Execute Ansible Role sap_ha_pacemaker_cluster
          ansible.builtin.include_role:
          o  name: community.sap_install.sap_ha_pacemaker_cluster
          vars:
            sap_ha_pacemaker_cluster_system_roles_collection: redhat.rhel_system_roles
            ha_cluster_cluster_name: clusternwasascs
            ha_cluster_hacluster_password: 'clusterpass'
            ha_cluster:
              node_name: "{{ ansible_hostname }}"
              pcs_address: "{{ ansible_default_ipv4.address }}"
            sap_ha_pacemaker_cluster_create_config_varfile: false
            sap_ha_pacemaker_cluster_host_type:
              - nwas_abap_ascs_ers
            sap_ha_pacemaker_cluster_vip_resource_group_name: vipnwasascs
            # Underlying filesystems are derived from the parent "/usr/sap" definition.
            sap_ha_pacemaker_cluster_storage_definition:
              - name: usr_sap
                mountpoint: /usr/sap
                nfs_path: /usr/sap
                nfs_server: "{{ sap_storage_nfs_server }}"

              - name: usr_sap_trans
                mountpoint: /usr/sap/trans
                nfs_path: /usr/sap/trans
                nfs_server: "{{ sap_storage_nfs_server }}"

              - name: sapmnt
                mountpoint: /sapmnt
                nfs_path: /sapmnt
                nfs_server: "{{ sap_storage_nfs_server }}"

            sap_ha_pacemaker_cluster_storage_nfs_filesytem_type: nfs4
            sap_ha_pacemaker_cluster_storage_nfs_mount_options: defaults,hard,acl
            sap_ha_pacemaker_cluster_storage_nfs_server: "{{ sap_storage_nfs_server | default('') }}"

            # SID and Instance Numbers for ASCS and ERS.
            sap_ha_pacemaker_cluster_nwas_abap_sid: "{{ sap_swpm_sid }}"
            sap_ha_pacemaker_cluster_nwas_abap_ascs_instance_nr: "{{ sap_swpm_ascs_instance_nr }}"
            sap_ha_pacemaker_cluster_nwas_abap_ers_instance_nr: "{{ sap_swpm_ers_instance_nr }}"

            # ASCS Profile name created by the installer, for example: <SID>_ASCS<Instance-Number>_<ASCS-virtual-node-name>
            sap_ha_pacemaker_cluster_nwas_abap_ascs_sapinstance_instance_name: "{{ sap_swpm_sid }}_ASCS{{ sap_swpm_ascs_instance_nr }}"
            sap_ha_pacemaker_cluster_nwas_abap_ascs_sapinstance_start_profile_string: "/sapmnt/{{ sap_swpm_sid }}/profile/{{ sap_swpm_sid }}_ASCS{{ sap_swpm_ascs_instance_nr }}_{{ sap_swpm_ascs_instance_hostname }}"

            # ERS Profile name created by the installer, for example: <SID>_ERS<Instance-Number>_<ERS-virtual-node-name>
            sap_ha_pacemaker_cluster_nwas_abap_ers_sapinstance_instance_name: "{{ sap_swpm_sid }}_ERS{{ sap_swpm_ers_instance_nr }}"
            sap_ha_pacemaker_cluster_nwas_abap_ers_sapinstance_start_profile_string: "/sapmnt/{{ sap_swpm_sid }}/profile/{{ sap_swpm_sid }}_ERS{{ sap_swpm_ers_instance_nr }}_{{ sap_swpm_ers_instance_hostname }}"
    ```

    ```bash
    [student@workstation ansible-files]$ ansible-playbook -v -K install-s4-ha-phase4*
    BECOME password: student
    ... output omitted ...
    ```

9. Create and execute the playbook install-s4-ha-phase5-dbload.yml
       - name:  Ansible Play for SAP NetWeaver Application Server - Installation Export Database Load from the Primary Application Server (PAS)
         hosts: s4pas
         become: true
         any_errors_fatal: true
         max_fail_percentage: 0
         tasks:
           - name: ensure software mountpoint exists
             ansible.builtin.file:
                path: "{{ sap_swpm_software_path }}"
                state: directory
                mode: '0755'
       
           - name: Ensure SAP software directory is mounted
             ansible.posix.mount:
               src: "utility:{{ sap_swpm_software_path }}"
               path: "{{ sap_swpm_software_path }}"
               opts: rw
               boot: no
               fstype: nfs
               state: mounted
       
           - name: Execute Ansible Role sap_swpm
             ansible.builtin.include_role:
               name: community.sap_install.sap_swpm
             vars:
               sap_swpm_templates_product_input: "sap_s4hana_2021_distributed_nwas_pas_dbload_ha"
               sap_swpm_virtual_hostname: "{{ sap_swpm_pas_instance_hostname }}"
       
           - name: Ensure SAP software directory is unmounted
             mount:
               path: "{{ sap_swpm_software_path }}"
               state: unmounted
       
 
        [student@workstation ansible-files]$ ansible-playbook -v -K install-s4-ha-phase5*
        BECOME password: student
        ... output omitted ...

10. Create and execute the playbook install-s4-ha-phase6-pas.yml
       - name:  Ansible Play for SAP NetWeaver Application Server - Installation Export Database Load from the Primary Application Server (PAS)
         hosts: s4pas
         become: true
         any_errors_fatal: true
         max_fail_percentage: 0
         tasks:
           - name: ensure software mountpoint exists
             ansible.builtin.file:
                path: "{{ sap_swpm_software_path }}"
                state: directory
                mode: '0755'
       
           - name: Ensure SAP software directory is mounted
             ansible.posix.mount:
               src: "utility:{{ sap_swpm_software_path }}"
               path: "{{ sap_swpm_software_path }}"
               opts: rw
               boot: no
               fstype: nfs
               state: mounted
       
           - name: Execute Ansible Role sap_swpm
             ansible.builtin.include_role:
               name: community.sap_install.sap_swpm
             vars:
               sap_swpm_templates_product_input: "sap_s4hana_2021_distributed_nwas_pas"
               sap_swpm_virtual_hostname: "{{ sap_swpm_pas_instance_hostname }}"
       
           - name: Ensure SAP software directory is unmounted
             mount:
               path: "{{ sap_swpm_software_path }}"
               state: unmounted
       
 
        [student@workstation ansible-files]$ ansible-playbook -v -K install-s4-ha-phase6*
        BECOME password: student
        ... output omitted ...

11. Create and execute the playbook install-s4-ha-phase7-aas.yml
       - name:  Ansible Play for SAP NetWeaver AAS
         hosts: s4aas
         become: true
         any_errors_fatal: true
         max_fail_percentage: 0
         tasks:
           - name: ensure software mountpoint exists
             ansible.builtin.file:
                path: "{{ sap_swpm_software_path }}"
                state: directory
                mode: '0755'
       
           - name: Ensure SAP software directory is mounted
             ansible.posix.mount:
               src: "utility:{{ sap_swpm_software_path }}"
               path: "{{ sap_swpm_software_path }}"
               opts: rw
               boot: no
               fstype: nfs
               state: mounted
       
           - name: Execute Ansible Role sap_swpm
             ansible.builtin.include_role:
               name: community.sap_install.sap_swpm
             vars:
               sap_swpm_templates_product_input: "sap_s4hana_2021_distributed_nwas_aas"
               sap_swpm_virtual_hostname: "{{ sap_swpm_aas_instance_hostname }}"
       
           - name: Ensure SAP software directory is unmounted
             mount:
               path: "{{ sap_swpm_software_path }}"
               state: unmounted
       
 
        [student@workstation ansible-files]$ ansible-playbook -v -K install-s4-ha-phase7*
        BECOME password: student
        ... output omitted ...

12.  Verify that SAP S/4 is running on the PAS:

        [student@workstation ~]$ ssh nodec
        [student@nodec ~]$ sudo su - rheadm
        [sudo] password for student: student
        Last login: Di Sep  6 11:38:52 EDT 2022 on pts/0
        nodea:rheadm 5> R3trans -d rheadm
        This is R3trans version 6.26 (release 785 - 25.11.21 - 11:06:00 including rjh702 ).
        unicode enabled version
        R3trans finished (0000).

    If the return value is 0, then everything is fine.

## Finish

You have successfully installed a distributed SAP S/4HANA Foundation with pacemaker cluster.

- Run the `lab` command on the `workstation` machine, and use the
  `lab` command to create the files in this exercise.
