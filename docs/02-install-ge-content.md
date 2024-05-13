---
layout: default
title: Verifying Ansible Configuration on the Control Node
nav_order: 2
has_children: true
permalink: /
---

# Guided Exercise: Verifying Ansible Configuration on the Control Node

In this exercise, you access and use the lab environment and browse the
available resources.

**Outcomes**

You verify the Ansible installation on the control node, and verify SSH
access to the managed nodes.

As the `student` user on the `workstation` machine, use the `lab`
command to prepare your system for this exercise.

This command ensures that the environment is configured correctly for
creating your Ansible Playbooks in the future.

    [student@workstation ~]$ lab start ansible-prereq

1.  Verify that Ansible 2.11 or later is installed and usable:

        [student@workstation ~]$ ansible --version
        ansible [core 2.12.2]
          config file = /etc/ansible/ansible.cfg
          configured module search path = ['/home/student/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
          ansible python module location = /usr/lib/python3.8/site-packages/ansible
          ansible collection location = /home/student/.ansible/collections:/usr/share/ansible/collections
          executable location = /usr/bin/ansible
          python version = 3.8.12 (default, Sep 16 2021, 10:46:05) [GCC 8.5.0 20210514 (Red Hat 8.5.0-3)]
          jinja version = 2.10.3
          libyaml = True

2.  Verify that Ansible inventory setup for this environment is correct.

    1.  Verify that you can see all hosts:

            [student@workstation ~]$ ansible all --list-hosts
              hosts (6):
                hana1.lab.example.com
                hana2.lab.example.com
                nodea.lab.example.com
                nodeb.lab.example.com
                nodec.lab.example.com
                noded.lab.example.com

    2.  Use `ansible ping` to test the connections from your management
        hosts to your managed nodes:

            [student@workstation ~]$ ansible -m ping all
            nodec.lab.example.com | SUCCESS => {
                "ansible_facts": {
                    "discovered_interpreter_python": "/usr/libexec/platform-python"
                },
                "changed": false,
                "ping": "pong"
            }
            hana1.lab.example.com | SUCCESS => {
                "ansible_facts": {
                    "discovered_interpreter_python": "/usr/libexec/platform-python"
                },
                "changed": false,
                "ping": "pong"
            }
            nodeb.lab.example.com | SUCCESS => {
                "ansible_facts": {
                    "discovered_interpreter_python": "/usr/libexec/platform-python"
                },
                "changed": false,
                "ping": "pong"
            }
            nodea.lab.example.com | SUCCESS => {
                "ansible_facts": {
                    "discovered_interpreter_python": "/usr/libexec/platform-python"
                },
                "changed": false,
                "ping": "pong"
            }
            hana2.lab.example.com | SUCCESS => {
                "ansible_facts": {
                    "discovered_interpreter_python": "/usr/libexec/platform-python"
                },
                "changed": false,
                "ping": "pong"
            }
            noded.lab.example.com | SUCCESS => {
                "ansible_facts": {
                    "discovered_interpreter_python": "/usr/libexec/platform-python"
                },
                "changed": false,
                "ping": "pong"
            }

**Finish**

On the `workstation` machine, use the `lab` command to complete this
exercise. This step is important to ensure that resources from previous
exercises do not impact upcoming exercises.

    [student@workstation ~]$ lab finish ansible-prereq
