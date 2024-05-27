---
layout: default
title: Preparing Ansible Playbooks to Configure SAP Servers for SAP Installation
nav_order: 5
parent: Day 1
---

# Guided Exercise: Preparing Ansible Playbooks to Configure SAP Servers for SAP Installation

In this exercise, you access and use the lab environment, and browse the
available resources.

## Outcomes

You write a playbook so that all servers are ready to consume SAP HANA
or SAP NetWeaver-based software.

<!--
As the `student` user on the `workstation` machine, execute the
following commands to download the SAP software to the training
environment:

    [student@workstation ~]$ git clone https://github.com/redhat-sap/RH445-download
    [student@workstation ~] cd RH445-download
    [student@workstation RH445-download] ./download-sap-media.sh

Follow the instructions on the screen.
-->

As the `student` user on the `workstation` machine, use the `lab`
command to prepare your system for this exercise.

This command ensures that the environment is configured correctly for
creating your Ansible Playbooks in the following lab.

{% raw %}

```bash
[student@workstation ~]$ lab start sap-baseconfig
```

{% endraw %}

In this course, you do not configure separate disks or a network in the
lab environment, although it is common in a production environment.

1.  Change to the `ansible-files` directory in your home directory:

    ```bash
    [student@workstation ~]$ cd ~/ansible-files
    ```

2.  Create the `group_vars/hanas` file with the following content:

    {% raw %}

    ```yaml
    ### If the hostname setup is not configured correctly
    #   you need set sap_ip and sap_domain.
    #   we use the full qualified domain name in the inventory
    #   so we can generate thes variables
    sap_hostname: "{{ inventory_hostname.split('.')[0] }}"
    sap_domain: "{{ inventory_hostname.split('.')[1:]| join('.') }}"

    ### redhat.sap_install.sap_general_preconfigure
    sap_general_preconfigure_modify_etc_hosts: true
    sap_general_preconfigure_fail_if_reboot_required: false
    sap_general_preconfigure_update: true
    sap_general_preconfigure_system_roles_collection: 'redhat.rhel_system_roles'

    ### redhat.sap_install.sap_hana_preconfigure
    sap_hana_preconfigure_update: true
    sap_hana_preconfigure_fail_if_reboot_required: false
    sap_hana_preconfigure_reboot_ok: true
    sap_hana_preconfigure_system_roles_collection: 'redhat.rhel_system_roles'
    ```

    {% endraw %}

    With this configuration, you achieve the following outcomes:

    - You update name resolution according to SAP requirements on each
      server.

    - The playbook does not stop if a reboot is required.

    - You update the system and reboot if required at the end of the
      `sap_hana_preconfigure` role.

    - The roles from `community.sap_install` are forced to use the supproted `redhat.rhel_system_roles` instead of the unsupported `fedora.linux_system_roles`

3.  Create the `group_vars/s4hanas` file with the following content:

    {% raw %}

    ```yaml
    ### If the hostname setup is not configured correctly
    #   you need set sap_ip and sap_domain.
    #   we use the full qualified domain name in the inventory
    #   so we can generate thes variables
    sap_hostname: "{{ inventory_hostname.split('.')[0] }}"
    sap_domain: "{{ inventory_hostname.split('.')[1:]| join('.') }}"

    ### redhat.sap_install.sap_general_preconfigure
    sap_general_preconfigure_modify_etc_hosts: true
    sap_general_preconfigure_update: true
    sap_general_preconfigure_fail_if_reboot_required: false
    sap_general_preconfigure_reboot_ok: true
    sap_general_preconfigure_system_roles_collection: 'redhat.rhel_system_roles'

    ### redhat.sap_install.sap_netweaver_preconfigure
    sap_netweaver_preconfigure_fail_if_not_enough_swap_space_configured: false
    sap_hana_preconfigure_system_roles_collection: 'redhat.rhel_system_roles'
    ```

    {% endraw %}

    With this configuration, you achieve the following outcomes:

    - You update DNS and the `/etc/host` file according to SAP
      requirements.

    - The playbook does not stop if a reboot is required.

    - You update the system and reboot if required.

    - Because you are installing a small lab system, you do not need
      to add a large swap space.

    - The roles from `community.sap_install` are forced to use the supproted `redhat.rhel_system_roles` instead of the unsupported `fedora.linux_system_roles`

4.  Create the `prepare-for-sap.yml` playbook to prepare the HANA and
    NetWeaver servers:

    {% raw %}

    ```yaml
    ---
    - name: Phase 3A - prepare for SAP HANA installation
      hosts: hanas
      become: true

      roles:
          - community.sap_install.sap_general_preconfigure
          - community.sap_install.sap_hana_preconfigure

    - name: Phase 4A - prepare for SAP NetWeaver installation
      hosts: s4hanas
      become: true
      roles:
          - community.sap_install.sap_general_preconfigure
          - community.sap_install.sap_netweaver_preconfigure
    ```

    {% endraw %}

5. In the current version of the lab we have an outdated package repository. So we need to remove the compatt-sap-c++-11 requirement from the latest role, because that was not available a year ago. To so run the following command:

    ```bash
    sed -i 's/  - compat-sap-c++-11/  # compat-sap-c++-11/g' /home/student/.ansible/collections/ansible_collections/community/sap_install/roles/sap_general_preconfigure/vars/RedHat_8.yml
    ```

6.  Execute the `prepare-for-sap.yml` playbook:

    ```bash
    [student@workstation ansible-files]$ ansible-playbook prepare-for-sap.yml -v -K
    BECOME password: student
    ...output omitted...
    ```

    A description of the parameters:

    - `-v`: Enables debugging (more verbose output)

    - `-K`: Prompts for the `sudo` password

**Finish**

To complete this exercise, take these steps:

- Run the `lab` command on the `workstation` machine, to create the
  files in this exercise.

- Run the `ansible-playbook` command to prepare the servers for SAP
  installation if not successful previously, and complete the
  exercise.

<!--

    [student@workstation ansible-files]$ lab finish sap-baseconfig
    [student@workstation ansible-files]$ ansible-playbook prepare-for-sap.yml -v -K
    BECOME password: student

--->