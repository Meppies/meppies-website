---
layout: post
title: Upgrade Citrix CVAD Database with Ansible code
Date: 2024-02-05
categories: [Citrix, CVAD, Ansible, Powershell]
thumbnail: "assets/images/database_upgrade.png"
description: To upgrade the on-premises database of your Citrix CVAD environment, you have two options. You can either perform the upgrade manually, following a step-by-step process, or you can automate the entire procedure using code. Opting for the latter provides you with greater control and allows you to schedule the upgrade according to your preferred timeline.
---

## Ansible
I used Ansible to automate the upgrade process, leveraging variables (denoted by {% raw %}{{ }}{% endraw %}) to enhance flexibility. All commands are PowerShell-based, adaptable to your preferred automation tool.


### Backup the databases

Ensure a comprehensive backup of all relevant databases is performed before initiating the upgrade.
{% raw %}
```yaml
- name: Backup database by the first Citrix Delivery Controller
    win_shell: Backup-SqlDatabase -ServerInstance "{{ sql_server }}" -Database "{{ item }}" -BackupFile "{{ citrix.database.backup_path }}\\{{ item }}_{{ lookup('pipe', 'date +%Y%m%d') }}.bak" 
    with_items:
        - "{{ citrix.database.site }}"
        - "{{ citrix.database.logging }}"
        - "{{ citrix.database.monitoring }}"
    vars:
        ansible_become: yes
        ansible_become_user: "{{ ansible_user }}"
        ansible_become_password: "{{ ansible_password }}"
        ansible_become_method: runas
```
{% endraw %}

### Monitoring Database script

Retrieve the necessary script to upgrade the monitoring database. This process includes determining the latest compatible version and requesting the appropriate upgrade script.
{% raw %}
```yaml
- name: Generate Citrix monitoring database upgrade scripts
    win_shell: |
        $version = Get-MonitorInstalledDBVersion -Upgrade
        if ($version) {
        $sql = Get-MonitorDBVersionChangeScript -DatabaseName "{{ citrix.database.monitoring }}" -TargetVersion "$($version.Major[0]).$($version.Minor[0]).$($version.Build[0]).$($version.Revision[0])"
        return $($sql.script)
        }
    register: mon_upgrade
    vars:
        ansible_become: yes
        ansible_become_user: "{{ ansible_user }}"
        ansible_become_password: "{{ ansible_password }}"
        ansible_become_method: runas
```
{% endraw %}

### Logging Database script

We will now repeat the upgrade process, this time focusing on the Logging database.
{% raw %}
```yaml
- name: Generate Citrix logging database upgrade scripts
    win_shell: |
        $version = Get-LogInstalledDBVersion -Upgrade
        if ($version) {
        $sql = Get-LogDBVersionChangeScript -DatabaseName "{{ citrix.database.logging }}" -TargetVersion "$($version.Major[0]).$($version.Minor[0]).$($version.Build[0]).$($version.Revision[0])"
        return $($sql.script)
        }
    register: log_upgrade
    vars:
        ansible_become: yes
        ansible_become_user: "{{ ansible_user }}"
        ansible_become_password: "{{ ansible_password }}"
        ansible_become_method: runas
```
{% endraw %}

### Site Database script

Generate the required scripts to upgrade the site database. This involves creating multiple scripts that must be run sequentially to complete the upgrade process.
{% raw %}
```yaml
- name: Generate Citrix site database upgrade scripts
    win_shell: |
        $version = {{ item.check }} -Upgrade
        if ($version) {
            $sql = {{ item.command }} -DatabaseName "{{ citrix.database.site }}" -TargetVersion "$($version.Major[0]).$($version.Minor[0]).$($version.Build[0]).$($version.Revision[0])"
            return $($sql.script)
        }
    loop:
        - { check: 'Get-AcctInstalledDBVersion', command: 'Get-AcctDBVersionChangeScript'}
        - { check: 'Get-AnalyticsInstalledDBVersion', command: 'Get-AnalyticsDBVersionChangeScript'}
        - { check: 'Get-AppLibInstalledDBVersion', command: 'Get-AppLibDBVersionChangeScript'}
        - { check: 'Get-ConfigInstalledDBVersion', command: 'Get-ConfigDBVersionChangeScript'}
        - { check: 'Get-AdminInstalledDBVersion', command: 'Get-AdminDBVersionChangeScript'}
        - { check: 'Get-EnvTestInstalledDBVersion', command: 'Get-EnvTestDBVersionChangeScript'}
        - { check: 'Get-HypInstalledDBVersion', command: 'Get-HypDBVersionChangeScript'}
        - { check: 'Get-ProvInstalledDBVersion', command: 'Get-ProvDBVersionChangeScript'}
        - { check: 'Get-OrchInstalledDBVersion', command: 'Get-OrchDBVersionChangeScript'}
        - { check: 'Get-SfInstalledDBVersion', command: 'Get-SfDBVersionChangeScript'}
        - { check: 'Get-TrustInstalledDBVersion', command: 'Get-TrustDBVersionChangeScript'}
        - { check: 'Get-BrokerInstalledDbVersion', command: 'Get-BrokerDBVersionChangeScript'}
    register: site_upgrade
    vars:
        ansible_become: yes
        ansible_become_user: "{{ ansible_user }}"
        ansible_become_password: "{{ ansible_password }}"
        ansible_become_method: runas
```
{% endraw %}

### Create a directory to save the scripts

Because the SQL scripts are to long for the powershell command you need to save the Scripts to a file.
{% raw %}
```yaml
- name: Create directory structure
    win_file:
        path: "{{ citrix.backup }}"
        state: directory
```
{% endraw %}

### Save the Scripts

Save the scripts to a file for future execution.
{% raw %}
```yaml
- name: Generate the SQL scripts on the SQL server
    ansible.windows.win_copy:
        content: "{{ item.stdout | trim }}"
        dest: "{{ citrix.backup }}\\upgrade_{{index}}.sql"
    when: item.stdout != ""
    with_items: 
        - "{{ site_upgrade.results }}"
        - "{{ mon_upgrade.results }}"
        - "{{ log_upgrade.results }}"
    loop_control:
    index_var: index
```
{% endraw %}

### Stop Citrix the services

To ensure a smooth database upgrade process, it is necessary to halt all Delivery Controller services. This will temporarily disrupt user access to published applications and desktops in StoreFront.
{% raw %}
```yaml
- name: stop citrix services
    win_shell: get-service citrix* -computername {{ item }} | stop-service -force
    loop: "{{ groups['CitrixDeliveryController'] }}"
    vars:
        ansible_become: yes
        ansible_become_user: "{{ ansible_user }}"
        ansible_become_password: "{{ ansible_password }}"
        ansible_become_method: runas
```
{% endraw %}

### Start the upgrade of the databases

The database upgrade will commence with the Site database. Once completed, we will proceed with upgrading the other necessary databases.
{% raw %}
```yaml
- name: Update the schema of the Citrix Delivery Controller databases
    win_shell: Invoke-Sqlcmd -ServerInstance "{{ sql_server }}" -InputFile "{{ citrix.backup }}\\upgrade_{{index}}.sql" 
    when: item.stdout != ""
    with_items:
        - "{{ site_upgrade.results }}"
        - "{{ mon_upgrade.results }}"
        - "{{ log_upgrade.results }}"
    loop_control:
    index_var: index
    vars:
        ansible_become: yes
        ansible_become_user: "{{ ansible_user }}"
        ansible_become_password: "{{ ansible_password }}"
        ansible_become_method: runas
```
{% endraw %}

### Start the Citrix services 

The database upgrade process has been successfully finalized. You now restart the Delivery Controller services, restoring user access to published applications and desktops via StoreFront.
{% raw %}
```yaml
- name: start citrix services
    win_shell: get-service citrix* -computername {{ item }} | start-service
    loop: "{{ groups['CitrixDeliveryController'] }}"
    vars:
        ansible_become: yes
        ansible_become_user: "{{ ansible_user }}"
        ansible_become_password: "{{ ansible_password }}"
        ansible_become_method: runas
```
{% endraw %}

### Cleanup the files

Clean up the directory where the SQL scripts are stored by deleting any temporary files.
{% raw %}
```yaml
- name: Remove directory structure
    win_file:
        path: "{{ citrix.backup }}"
        state: absent
```
{% endraw %}