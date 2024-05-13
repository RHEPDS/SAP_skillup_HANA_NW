---
layout: default
title: Verifying and Installing Ansible Collections for SAP Installation
nav_order: 4
has_children: true
permalink: /
---

# Guided Exercise: Verifying and Installing Ansible Collections for SAP Installation

In this exercise, you install the collections from Ansible Galaxy that
you need to prepare and install SAP software.

**Outcomes**

You install the necessary Ansible collections to deploy a RHEL systems
landscape for SAP HANA, SAP S/4HANA, or NetWeaver.

As the `student` user on the `workstation` machine, use the `lab`
command to prepare your system for this exercise.

This command ensures that the environment is configured correctly for
creating your Ansible Playbooks in the future.

    [student@workstation ~]$ lab start ansible-labcheck

1.  Create a working directory structure for playbooks and configuration
    files:

        [student@workstation ~]$ mkdir ansible-files
        [student@workstation ~]$ cd ansible-files
        [student@workstation ansible-files]$ mkdir collections host_vars group_vars

2.  Install the required collections for this course.

    1.  Verify that the Red Hat supported collections are already
        installed on your system.

            [student@workstation ansible-files]$ ansible-galaxy collection list

            # /usr/share/ansible/collections/ansible_collections
            Collection               Version
            ------------------------ -------
            ansible.posix            1.4.0
            community.general        5.5.0
            redhat.rhel_system_roles 1.16.2
            redhat.sap_install       1.0.2

    2.  Create a file named `requirements.yml` in the
        `${HOME}/ansible-files/collections` directory, with the
        following content:

            ---
            collections:
                    - name: community.sap_libs
                      version: 1.1.0
                    - name: community.sap_install
                      version: 1.2.3
                    - name: community.sap_operations
                      version: 0.9.0
                    - name: https://github.com/sap-linuxlab/community.sap_launchpad.git
                      type: git

    3.  Install the collections that are named in this file:

            [student@workstation ansible-files]$ ansible-galaxy collection install \
            > -r collections/requirements.yml

            Cloning into '/home/student/.ansible/tmp/ansible-local-7270kh4rk3yg/tmpb_i1nhkj/community.sap_launchpaduj699pgn'...
            remote: Enumerating objects: 35, done.
            remote: Counting objects: 100% (35/35), done.
            remote: Compressing objects: 100% (29/29), done.
            remote: Total 35 (delta 2), reused 22 (delta 1), pack-reused 0
            Receiving objects: 100% (35/35), 27.98 KiB | 1.08 MiB/s, done.
            Resolving deltas: 100% (2/2), done.
            Your branch is up to date with 'origin/main'.
            Starting galaxy collection install process
            Process install dependency map
            Starting collection install process
            [...]
            Installing 'community.sap_launchpad:1.0.0' to '/home/student/.ansible/collections/ansible_collections/community/sap_launchpad'
            Created collection for community.sap_launchpad:1.0.0 at /home/student/.ansible/collections/ansible_collections/community/sap_launchpad
            community.sap_launchpad:1.0.0 was installed successfully

        This step installs the collections in the user's home directory.
        To install them globally, you must run this command as root and
        add to the command the `-p /usr/share/ansible/collections`
        parameter.

3.  Verify that all required collections are installed now:

        [student@workstation ansible-files]$ ansible-galaxy collection list

        Collection               Version
        ------------------------ -------
        ansible.posix            1.4.0
        community.general        5.5.0
        redhat.rhel_system_roles 1.16.2
        redhat.sap_install       1.1.0

        # /home/student/.ansible/collections/ansible_collections
        Collection               Version
        ------------------------ -------
        community.sap_install    1.2.3
        community.sap_launchpad  1.0.0
        community.sap_libs       1.1.0
        community.sap_operations 0.9.0

    This course uses Ansible without a dedicated Ansible Execution
    Environment (EE). If you use Ansible Execution Environments or
    Ansible Automation Platform, then you can use `ansible-builder` to
    create an EE that includes these collections.

**Finish**

On the `workstation` machine, use the `lab` command to complete this
exercise. This step is important to ensure that resources from previous
exercises do not impact upcoming exercises.

    [student@workstation ~]$ lab finish ansible-labcheck
