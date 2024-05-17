---
layout: default
title: Verifying Ansible Configuration on the Control Node
nav_order: 2
parent: Day 1
---

<!-- trunk-ignore(markdownlint/MD025) --># Guided Exercise: Verifying Ansible Configuration on the Control Node

In this exercise, you access and use the lab environment and browse the
available resources.

## Outcomes

You verify the Ansible installation on the control node, and verify SSH
access to the managed nodes.

As the `student` user on the `workstation` machine, use the `lab`
command to prepare your system for this exercise.

This command ensures that the environment is configured correctly for
creating your Ansible Playbooks in the future.

    [student@workstation ~]$ lab start ansible-prereq

To update the lab environment to the latest working release, please do the following:

As the `student` user on the `workstation` machine, execute the
following commands to download the SAP software to the training
environment:

    [student@workstation ~]$ git clone https://github.com/redhat-sap/RH445-download
    [student@workstation ~] cd RH445-download/tools
    [student@workstation tools] ./update-lab.sh
    ===============
    Warning: [...]
    ===============
    Press Ctrl-C ...
    BECOME password: student
    Activation Key: ask-your-trainer
    Enter a Nickname: anything here

1. Verify that Ansible 2.14 or later is installed and usable:

        [student@workstation ~]$ ansible --version
        ansible [core 2.15.3]
          config file = /home/student/.ansible.cfg
          configured module search path = ['/home/student/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
          ansible python module location = /usr/lib/python3.11/site-packages/ansible
          ansible collection location = /home/student/.ansible/collections:/usr/share/ansible/collections
          executable location = /usr/bin/ansible
          python version = 3.11.5 (main, Sep 22 2023, 15:34:29) [GCC 8.5.0 20210514 (Red Hat 8.5.0-20)] (/usr/bin/python3.11)
          jinja version = 3.1.2
          libyaml = True

2. Verify that Ansible inventory setup for this environment is correct.

    1. Verify that you can see all hosts:

            [student@workstation ~]$ ansible all --list-hosts
              hosts (6):
                hana1.lab.example.com
                hana2.lab.example.com
                nodea.lab.example.com
                nodeb.lab.example.com
                nodec.lab.example.com
                noded.lab.example.com

    2.  Use `ansible ping` to test the connections from your management hosts to your managed nodes:

        {% raw %}
        ```bash
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
        ```
        {% endraw %}

## Dowload the SAP Software into your environment

We must not provide the SAP software installer bundles.
Hence you need an SAP S-User with download permission to download the SAP software to the training enviroment. The download is automated for you.

As the `student` user on the `workstation` machine, execute the
following commands to download the SAP software to the training
environment:

    [student@workstation ~] cd ~/RH445-download
    [student@workstation RH445-download] ./download-sap-media.sh

Follow the instructions on the screen and enter your S-User and password.

If you do not have an S-User/password to download the installer software.
The above script lists the files you need to provide to the course environment.
These files have to be provided on a server which can be reached from inside the course environment.

## Finish

On the `workstation` machine, use the `lab` command to complete this
exercise. This step is important to ensure that resources from previous
exercises do not impact upcoming exercises.

    [student@workstation ~]$ lab finish ansible-prereq
