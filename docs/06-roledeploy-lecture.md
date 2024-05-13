---
layout: default
title: Describing the SAP HANA Deployment
nav_order: 6
---

# Describing the SAP HANA Deployment

## Overview of the SAP HANA Deployment

In a previous section, you were introduced to the preparation of systems
to serve SAP software. Remember that all SAP software relies on a
database, in particular SAP HANA for the newer developments from SAP.
You must decide whether HANA is highly available or just stand-alone, in
a scale-up or a scale-out environment. This course focuses on the
commonest setup, a clustered SAP HANA scale-up configuration.

To create a clustered SAP HANA scale-up system, you must deploy SAP HANA
identically on two nodes. You must then set up the HANA System
Replication (HSR). When it is done, you can install and configure the
cluster.

To focus on the first step:

## Important Parameters for the SAP HANA Installation

In this section, the most important parameters for the SAP HANA
installation are explained. You can use a more granular configuration of
each role, which you can find in the documentation, or see the default
variable file at
[](https://github.com/sap-linuxlab/community.sap_install/blob/main/roles/sap_hana_install/defaults/main.yml).

### sap_hana_install

This role installs SAP HANA. It creates the configuration file for an
unattended installation of SAP HANA with `hdblcm` and starts the
installation process.

The minimum parameters to set for this role are as follows:

- `sap_hana_install_software_directory`: The directory with the
  unpacked HANA software or the downloaded archive

- `sap_hana_install_master_password`: The password for the SAP
  administrative user

- `sap_hana_sid`: The 3-character SID of the HANA database

- `sap_hana_instance_number`: The instance number of the HANA database

If the archives are on a shared file server, you must set add the
`sap_hana_install_software_extract_directory` variable to a local
directory to avoid race conditions when installing HANA in parallel on
multiple hosts.

The role expects the `SAPCAR*.EXE` file in the directory that is defined
in the `sap_hana_install_software_directory` variable. If the file is
placed somewhere else, then you must define the
`sap_hana_install_sapcar_filename` variable.

In production environments, define password variables in an Ansible
vault file or as custom credentials in Ansible Controller.

## Additional Information

[Definition of Race
Condition](https://en.wikipedia.org/wiki/Race_condition#In_software)

[A Brief Introduction to Ansible
Vault](https://www.redhat.com/sysadmin/introduction-ansible-vault)

[How to Encrypt Sensitive Data in Playbooks with Ansible
Vault](https://www.redhat.com/sysadmin/ansible-vault-secure-playbooks)

[Ansible Controller Custom Credential
Types](https://docs.ansible.com/automation-controller/latest/html/userguide/credential_types.html)

[sap_hana_install Ansible Role
documentation](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/sap_install/content/role/sap_hana_install)

[Overview of SAP HANA Configuration File
Parameters](https://bit.ly/sap-hana-config-file-params)
