---
layout: default
title: Introducing Roles for SAP S/4 Deployments
nav_order: 20
---

# Introducing Roles for SAP S/4 Deployments

## Overview of Installation of S/4 HANA

SAP Applications will be installed with the SAP Software Provisioning
Manager (SAP SWPM). The `sap_swpm` role creates a configuration file for
the SAP SWPM software for unattended installation. The role can install
everything that SAP SWPM (or the `sapinst` command) can install. It is
tested and working for the following scenarios:

- One host installation
- Dual host installation
- Distributed installation
- System restore
- High availability installation

The role is tested and working for the following SAP products:

- SAP S/4HANA 1909, 2020, 2021
- SAP B/4HANA
- SAP Solution Manager 7.2
- SAP NetWeaver Business Suite Applications (ECC, GRC, and so on)
- SAP Web Dispatcher

The role is currently available only in the upstream repository, in the
`community.sap_install collection`.

To download the installation bundle, which is needed for your
installation, you could use the _SAP Maintenance Planner_,
[](https://userapps.support.sap.com/sap/support/mp/index.html).

The SAP Maintenance Planner helps you to create a list of all
installation files for your chosen SAP software. Then, you must download
this list of files from the SAP Software Download Center, and place it
in a directory. The `sap_swpm` role can then install the software in
that directory.

## Parameters of the `sap_swpm` Role

This course focuses on the needed parameters for SAP S/4 HANA. The
following table defines the variables for the default installation
method of SAP HANA, which is tested and works for S/4HANA 2020 and
S/4HANA 2021.

### Parameters for Software Installation

<table>
<colgroup>
<col style="width: 40%" />
<col style="width: 60%" />
</colgroup>
<thead>
<tr class="header">
<th style="text-align: left;">Parameter</th>
<th style="text-align: left;">Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td
style="text-align: left;"><code>sap_swpm_product_catalog_id</code></td>
<td style="text-align: left;">The mandatory parameter that defines which
software SWPM installs. Examples are as follows:<br />
Single host installation of S4HANA 2021 Foundation:<br />
<em>NW_ABAP_OneHost:S4HANA2021.FNDN.HDB.ABAP</em><br />
Single host installation of S4HANA 2021 full ERP system:<br />
<em>NW_ABAP_OneHost:S4HANA2021.CORE.HDB.ABAP</em></td>
</tr>
<tr class="even">
<td style="text-align: left;"><code>sap_swpm_update_etchosts</code></td>
<td style="text-align: left;">Whether to update the
<code>/etc/hosts</code> file (default: true)</td>
</tr>
<tr class="odd">
<td style="text-align: left;"><code>sap_swpm_software_path</code></td>
<td style="text-align: left;">Path to the downloaded software bundle
(currently must be writable, because the bundle files are unpacked
here)</td>
</tr>
<tr class="even">
<td style="text-align: left;"><code>sap_swpm_sapcar_path</code></td>
<td style="text-align: left;">Path to the directory that contains the
<code>sapcar</code> utility (looks for the <code>SAPCAR*.EXE</code>
pattern)</td>
</tr>
<tr class="odd">
<td style="text-align: left;"><code>sap_swpm_swpm_path</code></td>
<td style="text-align: left;">Path to the directory that contains
<code>SWPM*.SAR</code></td>
</tr>
</tbody>
</table>

### Passwords

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th style="text-align: left;">Parameter</th>
<th style="text-align: left;">Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: left;"><code>sap_swpm_master_password</code></td>
<td style="text-align: left;">Master password: Must contain uppercase,
lowercase, numbers, and special characters (must not contain
<code>!</code>)</td>
</tr>
<tr class="even">
<td
style="text-align: left;"><code>sap_swpm_ddic_000_password</code></td>
<td style="text-align: left;">DDIC password: Must contain uppercase,
lowercase, numbers, and special characters (must not contain
<code>!</code>)</td>
</tr>
<tr class="odd">
<td
style="text-align: left;"><code>sap_swpm_db_system_password</code></td>
<td style="text-align: left;">SAP HANA System password</td>
</tr>
<tr class="even">
<td
style="text-align: left;"><code>sap_swpm_db_systemdb_password</code></td>
<td style="text-align: left;">SAP HANA System Database password</td>
</tr>
<tr class="odd">
<td
style="text-align: left;"><code>sap_swpm_db_schema_abap_password</code></td>
<td style="text-align: left;">SAP HANA System Database ABAP Schema
password</td>
</tr>
<tr class="even">
<td
style="text-align: left;"><code>sap_swpm_db_sidadm_password</code></td>
<td style="text-align: left;">SAP HANA <code>sidadm</code> password</td>
</tr>
</tbody>
</table>

### NetWeaver Instance Parameters

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th style="text-align: left;">Parameter</th>
<th style="text-align: left;">Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: left;"><code>sap_swpm_sid</code></td>
<td style="text-align: left;">SID of the NetWeaver instance (such as
RHE)</td>
</tr>
<tr class="even">
<td style="text-align: left;"><code>sap_swpm_pas_instance_nr</code></td>
<td style="text-align: left;">Instance number of the primary application
server (such as <code>01</code>)</td>
</tr>
<tr class="odd">
<td
style="text-align: left;"><code>sap_swpm_ascs_instance_nr</code></td>
<td style="text-align: left;">Instance number of the central services
(such as <code>02</code>)</td>
</tr>
<tr class="even">
<td
style="text-align: left;"><code>sap_swpm_ascs_instance_hostname</code></td>
<td style="text-align: left;">Hostname where the ASCS runs (set to
<code>{{ ansible_hostname }}</code> on single-node installations)</td>
</tr>
<tr class="odd">
<td style="text-align: left;"><code>sap_swpm_fqdn</code></td>
<td style="text-align: left;">Domain name of the SAP installation, such
as <code>{{ sap_domain }}</code>. Note that SAP uses the <em>fqdn</em>
abbreviation for the domain name only.</td>
</tr>
</tbody>
</table>

### HANA Database Instance Parameters

If your HANA instance is running on a different system, you must define
the connection parameters to your HANA system:

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th style="text-align: left;">Parameter</th>
<th style="text-align: left;">Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td style="text-align: left;"><code>sap_swpm_db_host</code></td>
<td style="text-align: left;">Short hostname of your HANA instance. It
must be defined in the <code>/etc/hosts</code> file, or be properly
resolved by DNS.</td>
</tr>
<tr class="even">
<td style="text-align: left;"><code>sap_swpm_db_sid</code></td>
<td style="text-align: left;">SAP HANA SID (such as "RHE")</td>
</tr>
<tr class="odd">
<td style="text-align: left;"><code>sap_swpm_db_instance_nr</code></td>
<td style="text-align: left;">SAP HANA instance number (such as
"00")</td>
</tr>
</tbody>
</table>

### Advanced Installation Methods

If you need special parameters, or you want to replicate an existing
running system, you can also pass the configuration files that were
created during a previous manual installation, or you can create your
own configuration file.

You must set the following parameters:

<table>
<colgroup>
<col style="width: 50%" />
<col style="width: 50%" />
</colgroup>
<thead>
<tr class="header">
<th style="text-align: left;">Parameter</th>
<th style="text-align: left;">Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td
style="text-align: left;"><code>sap_swpm_ansible_role_mode</code></td>
<td style="text-align: left;">Installation mode of this role. Possible
values are <em>default, default_templates, advanced, advanced_templates,
inifile_reuse</em></td>
</tr>
<tr class="even">
<td
style="text-align: left;"><code>sap_swpm_inifile_custom_values_dictionary</code></td>
<td style="text-align: left;">When setting
<code>sap_swpm_ansible_role_mode</code> to <em>advanced</em>, this
variable must contain the configuration file and the product ID in the
third line as the last value. See the following example.</td>
</tr>
</tbody>
</table>

### Example for SAP HANA 1909 Single-Node Installation (Requires Advanced Mode)

    # sap_swpm
    #----------
    sap_swpm_ansible_role_mode: advanced
    sap_swpm_sapcar_path: "/software/SAPCAR"
    sap_swpm_software_path: "/software/S4HANA_installation"
    sap_swpm_swpm_path: "/software/S4HANA_installation"

    # Do not touch /etc/hosts
    sap_swpm_update_etchosts: false

    sap_swpm_master_password: "R3dh4t$123"

    sap_swpm_inifile_custom_values_dictionary:
       '# Custom Config file created for SAP Workshop': ''
       '# Product catalog ID': ''
       '# NW_ABAP_OneHost:S4HANA1909.CORE.HDB.ABAP': ''
       HDB_Schema_Check_Dialogs.schemaPassword:  "{{ sap_swpm_master_password }}"
       HDB_Schema_Check_Dialogs.validateSchemaName:  "false"
       NW_CI_Instance.ascsInstanceNumber:  ""
       NW_CI_Instance.ascsVirtualHostname :  ""
       NW_CI_Instance.ciInstanceNumber :  ""
       NW_CI_Instance.ciVirtualHostname :  ""
       NW_CI_Instance.scsVirtualHostname :  ""
       NW_DDIC_Password.ddic000Password :  ""
       NW_Delete_Sapinst_Users.removeUsers :  "true"
       NW_GetMasterPassword.masterPwd : "{{ sap_swpm_master_password }}"
       NW_GetSidNoProfiles.sid :  RHE
       NW_HDB_DB.abapSchemaName :  ""
       NW_HDB_DB.abapSchemaPassword :  "{{ sap_swpm_master_password }}"
       NW_HDB_DB.javaSchemaName :  ""
       NW_HDB_DB.javaSchemaPassword :  ""
       NW_HDB_getDBInfo.dbhost : "hana-{{ guid }}1.example.com"
       NW_HDB_getDBInfo.dbsid :  RHE
       NW_HDB_getDBInfo.instanceNumber :  '00'
       NW_HDB_getDBInfo.systemDbPassword :  "{{ sap_swpm_master_password }}"
       NW_HDB_getDBInfo.systemPassword :  "{{ sap_swpm_master_password }}"
       NW_HDB_getDBInfo.systemid :  RHE
       NW_Recovery_Install_HDB.extractLocation :  /usr/sap/RHE/HDB00/backup/data/DB_RHE
       NW_Recovery_Install_HDB.extractParallelJobs :  '30'
       NW_Recovery_Install_HDB.sidAdmName :  rheadm
       NW_Recovery_Install_HDB.sidAdmPassword :  "{{ sap_swpm_master_password }}"
       NW_SAPCrypto.SAPCryptoFile :  '{{ sap_swpm_software_path }}'
       NW_getFQDN.FQDN : ''
       NW_getFQDN.setFQDN :  "true"
       NW_getLoadType.loadType :  SAP
       archives.downloadBasket :  '{{ sap_swpm_software_path }}'
       hdb.create.dbacockpit.user :  "true"
       hostAgent.sapAdmPassword :  "{{ sap_swpm_master_password }}"
       nwUsers.sidadmPassword :  "{{ sap_swpm_master_password }}"

With the advanced method, you do not have to define any of the HDB or
NetWeaver instance variables. You manually define the complete
configuration file. With this method, do not change the first three
lines. Adapt only the Product ID. This ID is also obtained from the
configuration file.

##  

[Upstream sap_swpm Role
Description](https://github.com/sap-linuxlab/community.sap_install/blob/main/roles/sap_swpm/README.md)
