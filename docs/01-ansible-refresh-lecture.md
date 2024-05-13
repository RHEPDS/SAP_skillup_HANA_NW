---
layout: default
title: Describe Ansible Concepts
nav_order: 1
---

# Describing Ansible Concepts

## Introduction

For taking this training, it is assumed that you already have decent
Ansible knowledge. To refresh your knowledge, the following sections
briefly cover the concepts and key components of Ansible.

## The Control Node and Execution Environment

The host, where all Ansible files are stored, such as Ansible Playbooks,
inventory, and collections, is often referred to as the
**Control Node**.

The host or container that finally executes an action, such as to run a
playbook with the `ansible-playbook` command, is often referred to as
the **Execution Engine**. Frequently, including in this course, the
control node contains the execution engine.

In Ansible Automation Platform 2 and later Ansible releases, running the
`ansible-navigator` command on the control node can perform actions on
different execution engines that can be spread across multiple hosts.

## The Inventory

The inventory can be a file or a script that creates the content
dynamically on access. The output format of the script must be JSON.

The static inventory file lists and groups the hosts to manage in
`ini-style`. For example:

    [appserver]
    host1.example.com
    host2.example.com

    [database]
    host3.example.com
    host4.example.com

The inventory is also a place where you can define variables. A good
practice is to store only connection-based variables such as for the IP
address, connection user, method or port, if needed in the inventory
file.

## The Ansible Configuration Files

You can customize the behavior of Ansible by modifying settings in the
Ansible `ini-style` configuration file. Ansible selects its
configuration file from one of several possible locations on the control
node. See
[](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#ansible-configuration-settings).

It is recommended to create an `ansible.cfg` file in a directory from
which you run Ansible commands. This directory would also contain any
files that your Ansible project uses, such as the inventory and
playbooks. You can now create your configurations that differ from the
global configuration file (`/etc/ansible/ansible.cfg`), in the local
`ansible.cfg` file.

The following example shows a configuration file that creates an entry
to use a locally defined inventory instead of the default
`/etc/ansible/hosts` file:

    [defaults]
    inventory=/home/ansible/ansible-files/inventory

This inventory is now used without a need to provide the `-i` option.

### Ansible Playbooks

Playbooks are files which describe the intended configurations or steps
to implement on managed hosts. Playbooks can change lengthy, complex
administrative tasks into easily repeatable routines with predictable
and successful outcomes.

An analogy: When Ansible modules are the tools in your workshop, the
inventory is the materials and the playbooks are the instructions.

Playbooks are text files that are written in YAML format, and therefore
have the following needs:

- To start with three dashes (`---`)

- Proper indentation with spaces and **not** tabs

These concepts are important:

- **hosts**: The managed hosts to perform the tasks on

- **tasks**: The operations to perform by invoking Ansible modules and
  passing them the necessary options

- **become**: Privilege escalation in playbooks

The ordering of the contents within a playbook is important, because
Ansible executes plays and tasks in the order that they are presented.

A playbook should be **idempotent**, so if a playbook is run once to put
the hosts in the correct state, then it should be safe to run it a
second time and it should not change the hosts further. Most Ansible
modules are idempotent, so it is relatively easy to ensure that it is
the case.

Try to avoid the command, shell, and raw modules in playbooks. Because
these modules take arbitrary commands, it is easy to end up with
non-idempotent playbooks with these modules.

An example playbook looks as follows:

    ---
    - name: Apache server installed
      hosts: host1.example.com
      become: yes
      tasks:
      - name: latest Apache version installed
        yum:
          name: httpd
          state: latest

In this playbook:

- A name is given for the play.

- The host to run against and privilege escalation are configured.

- A task is defined and named, here by using the "yum" module with the
  needed options.

### Ansible Variables

Ansible supports variables to store values that can be used in
playbooks. Variables can be defined in various places, and have a clear
precedence. Ansible substitutes the variable with its value when a task
is executed.

Variables are referenced in playbooks by placing the variable name in
double curly braces.

    Here comes a variable {{ variable1 }}

The recommended practice is to define variables in files in the
`host_vars` and `group_vars` directories:

- For example, to define variables for a group named `servers` that is
  already defined in the inventory, create a YAML file named
  `group_vars/servers` with the variable definitions.

- To define variables specifically for a `host1.example.com` host,
  create the `host_vars/host1.example.com` file with the variable
  definitions.

Host variables take precedence over group variables. (More information
about precedence can be found in the documentation.)

### Ansible Facts

Ansible facts are variables that Ansible automatically discovers from a
managed host. Facts are pulled by the `setup` module, and contain useful
information that is stored into variables that administrators can reuse.
The setup module is run by default at the beginning of every play,
unless you define `gather_facts: no`.

### Ansible Conditionals

Ansible can use conditionals to execute tasks or plays when certain
conditions are met.

To implement a conditional, the `when` statement must be used, followed
by the condition to test. The condition is expressed by using one of the
available operators, for example for comparison:

<table>
<colgroup>
<col style="width: 14%" />
<col style="width: 85%" />
</colgroup>
<tbody>
<tr class="odd">
<td style="text-align: left;">==</td>
<td style="text-align: left;">Compares two objects for equality.</td>
</tr>
<tr class="even">
<td style="text-align: left;">!=</td>
<td style="text-align: left;">Compares two objects for inequality.</td>
</tr>
<tr class="odd">
<td style="text-align: left;">&gt;</td>
<td style="text-align: left;">True if the left side is greater than the
right side.</td>
</tr>
<tr class="even">
<td style="text-align: left;">&gt;=</td>
<td style="text-align: left;">True if the left side is greater than or
equal to the right side.</td>
</tr>
<tr class="odd">
<td style="text-align: left;">&lt;</td>
<td style="text-align: left;">True if the left side is less than the
right side.</td>
</tr>
<tr class="even">
<td style="text-align: left;">&lt; =</td>
<td style="text-align: left;">True if the left side is less than or
equal to the right side.</td>
</tr>
</tbody>
</table>

For more information, see the Jinja2 Template Designer documentation:
[](http://jinja.pocoo.org/docs/2.9/templates/)

### Ansible Handlers

Sometimes, when a task changes the system, a further task might need to
be run. For example, a change to a service's configuration file might
then require restarting the service for the changed configuration to
take effect.

Here, the Ansible handlers come into play. Handlers can be seen as
inactive tasks that are triggered only when explicitly invoked with the
`notify` statement.

As a an example, consider a playbook with the following attributes:

- It manages the Apache `httpd.conf` configuration file on all hosts
  in the `webserver` group.

- It restarts Apache when the file changes.

<!-- -->

    ---
    - name: manage httpd.conf
      hosts: webserver
      become: yes
      tasks:
      - name: Copy Apache configuration file
        copy:
          src: httpd.conf
          dest: /etc/httpd/conf/
        notify:
           - restart_apache
      handlers:
        - name: restart_apache
          service:
            name: httpd
            state: restarted

So, what is new here?

- The `notify` section calls the handler only when the copy task
  changed the file.

- The `handlers` section defines a task that is run only on
  notification.

### Ansible Templates

Ansible uses Jinja2 templating to modify files before they are
distributed to managed hosts. Jinja2 is one of the most-used template
engines for Python ([](http://jinja.pocoo.org/)).

When a template for a file is created, it can be deployed to the managed
hosts by using the `template` module, which supports the transfer of a
local file from the control node to the managed hosts.

As an example of using templates, you can create the following `motd`
file to contain host-specific data.

In your templates directory, create the `motd-facts.j2` template file:

    Welcome to {{ ansible_hostname }}.
    {{ ansible_distribution }} {{ ansible_distribution_version}}
    deployed on {{ ansible_architecture }} architecture.

The following playbook would deploy the template file after replacing
the variables with the current host values that the setup module
collected:

    ---
    - name: Fill motd file with host data
      hosts: host1.example.com
      become: yes
      tasks:
        - template:
            src: motd-facts.j2
            dest: /etc/motd
            owner: root
            group: root
            mode: 0644

### Ansible Roles

Compared with the previous brief example, a HANA database installation
might need many more tasks. To write a playbook with all these tasks can
be quickly become confusing or unclear. Similar to creating
parameterized functions in other programming languages, you can create
roles in Ansible that can execute several tasks and be parameterized
with variables.

### Ansible Collections

Ansible Content Collections are a distribution format for Red Hat
Ansible Automation Platform content that can include playbooks, roles,
modules, and plug-ins around specific topic areas.

Ansible Content Collections represent the new standard of distributing,
maintaining, and consuming automation. By combining multiple types of
Ansible Automation Platform content, flexibility and scalability are
improved.

A content collection is designed to have a consistent format for content
creators to ship bundles of modules, plug-ins, roles, and documentation
together, and for users to consume these pieces from a single place.

Ansible Content Collections enable Ansible Automation Platform users to
get up and running with curated content from certified partners and from
Red Hat.

This prepackaged content is sorted by content domain and requires less
upfront work to find and assemble different roles and modules.

### Ansible Collections for SAP

Together with other partners, Red Hat is developing a comprehensive set
of collections to install and manage SAP systems with Ansible. For more
details, browse the project homepage at
[](https://sap-linuxlab.github.io). The fully supported collections are
in [Automation
Hub](https://console.redhat.com/ansible/automation-hub/?page_size=10&view_type=null&keywords=SAPhttps://console.redhat.com/ansible/automation-hub/repo/published/redhat/sap_install)

##  

[Getting Started with
Ansible](https://www.redhat.com/sysadmin/getting-started-ansible)

[What Are Ansible Content
Collections?](https://www.redhat.com/en/technologies/management/ansible/ansible-content-collections)

[Ansible Documentation](https://docs.ansible.com/)

[SAP Installation Collection at Automation
Hub](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/sap_install)
