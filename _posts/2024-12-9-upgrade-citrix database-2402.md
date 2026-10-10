---
layout: post
title: Upgrade Citrix CVAD 2402 LTSR Database with Ansible code
Date: 2024-12-09
categories: [Citrix, CVAD, Ansible, Powershell]
thumbnail: "assets/images/database_upgrade.png"
description: Citrix Virtual Apps and Desktops (CVAD) 2402 LTSR introduced modifications to the database upgrade process.
---
## Database upgrade
 
Citrix Virtual Apps and Desktops (CVAD) 2402 LTSR introduced changes to the database upgrade process, requiring additional manual steps. In collaboration with Citrix, we successfully automated these steps to streamline the upgrade process. The following steps outline the automated approach.


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

The upgrade process for the monitoring component has been slightly modified, necessitating the creation of an additional upgrade script for the site database.

Retrieve the necessary script to upgrade the monitoring database. This process includes determining the latest compatible version and requesting the appropriate upgrade script.

{% raw %}
```yaml
- name: Generate Citrix monitoring database upgrade scripts
    win_shell: |
        $version = Get-MonitorInstalledDBVersion -Upgrade
        if ($version) {
            $sql = Get-MonitorDBVersionChangeScript -DatabaseName "{{ citrix.database.monitoring }}" -TargetVersion "$($version.Major[0]).$($version.Minor[0]).$($version.Build[0]).$($version.Revision[0])" -DataStore "{{ item }}"
            return $($sql.script)
        }
    loop:
        - Monitor
    register: mon_upgrade
    vars:
        ansible_become: yes
        ansible_become_user: "{{ ansible_user }}"
        ansible_become_password: "{{ ansible_password }}"
        ansible_become_method: runas

- name: Generate Citrix monitoring database upgrade scripts
    win_shell: |
        $version = Get-MonitorInstalledDBVersion -Upgrade
        if ($version) {
            $sql = Get-MonitorDBVersionChangeScript -DatabaseName "{{ citrix.database.site }}" -TargetVersion "$($version.Major[0]).$($version.Minor[0]).$($version.Build[0]).$($version.Revision[0])" -DataStore "{{ item }}"
            return $($sql.script)
        }
    loop:
        - Site
    register: mon_site_upgrade
    vars:
        ansible_become: yes
        ansible_become_user: "{{ ansible_user }}"
        ansible_become_password: "{{ ansible_password }}"
        ansible_become_method: runas
```
{% endraw %}

### Logging Database script

The upgrade process for the Logging component has been slightly modified, necessitating the creation of an additional upgrade script for the site database.

We will now repeat the upgrade process, this time focusing on the Logging database.
{% raw %}
```yaml
- name: Generate Citrix logging database upgrade scripts
    win_shell: |
        $version = Get-LogInstalledDBVersion -Upgrade
        if ($version) {
            $sql = Get-LogDBVersionChangeScript -DatabaseName "{{ citrix.database.logging }}" -TargetVersion "$($version.Major[0]).$($version.Minor[0]).$($version.Build[0]).$($version.Revision[0])" -DataStore "{{ item }}"
            return $($sql.script)
        }
    loop:
        - Logging
    register: log_upgrade
    vars:
        ansible_become: yes
        ansible_become_user: "{{ ansible_user }}"
        ansible_become_password: "{{ ansible_password }}"
        ansible_become_method: runas

- name: Generate Citrix logging database upgrade scripts
    win_shell: |
        $version = Get-LogInstalledDBVersion -Upgrade
        if ($version) {
            $sql = Get-LogDBVersionChangeScript -DatabaseName "{{ citrix.database.site }}" -TargetVersion "$($version.Major[0]).$($version.Minor[0]).$($version.Build[0]).$($version.Revision[0])" -DataStore "{{ item }}"
            return $($sql.script)
        }
    loop:
        - Site
    register: log_site_upgrade
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
        - "{{ mon_site_upgrade.results }}"
        - "{{ log_site_upgrade.results }}"
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
        - "{{ mon_site_upgrade.results }}"
        - "{{ log_site_upgrade.results }}"
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


### Final step
This step, developed in collaboration with Citrix, should be executed on all Delivery Controllers after completing the database upgrade.

{% raw %}
```powershell
Function CheckInstanceRegistered($instance, $registeredInstances)
{
    foreach ($registeredInstance in $registeredInstances)
    {
        if (
            ($registeredInstance.ServiceType -eq $instance.ServiceType) -and
            ($registeredInstance.InterfaceType -eq $instance.InterfaceType) -and
            ($registeredInstance.Address -eq $instance.Address) -and
            ($registeredInstance.Version -eq $instance.Version) -and
            ($registeredInstance.Binding -eq $instance.Binding) -and
            ($registeredInstance.ServiceAccount -eq $instance.ServiceAccount) -and
            ($registeredInstance.ServiceAccountSid -eq $instance.ServiceAccountSid) -and
            ($registeredInstance.ServiceGroupName -eq $instance.ServiceGroupName) -and
            ($registeredInstance.ServiceGroupUid -eq $instance.ServiceGroupUid)            
        )
        {
            return $true
        }  
    }
    
    return $false
}

Function GetUnregisteredInstances($instances)
{
    $unregisteredInstances = @()
    $registeredInstances=Get-ConfigRegisteredServiceInstance

    foreach ($instance in $instances)
    {
        $registered = CheckInstanceRegistered $instance $registeredInstances
        if ($registered -eq $false)
        {
            $unregisteredInstances += $instance
        }
    }
    return $unregisteredInstances
}

Function RegisterServiceInstances($serviceType)
{
    $cmdlet="Get-{0}ServiceInstance" -f $serviceType  
    $instances=Invoke-Expression $cmdlet

    $unregisteredInstances = GetUnregisteredInstances $instances
    if ($unregisteredInstances.Count -ne 0)
    {
        Write-Host "Register service instance $ServiceType..."

        foreach ($instance in $unregisteredInstances)
        {
            Write-Host $instance.Address

            if (($instance.ServiceType -eq "Orch") -and ($instance.InterfaceType -eq "RestApi"))
            {
                Get-ConfigRegisteredServiceInstance -ServiceType "Orch" -InterfaceType "RestApi" | Unregister-ConfigRegisteredServiceInstance
            }
        }           
        Register-ConfigServiceInstance -ServiceInstance $unregisteredInstances
    }
}

Function UpgradeServiceInstances()
{
    $serviceTypes=@("Config", "Admin", "Log", "Acct", "Analytics", "AppLib", "EnvTest", "Hyp", "Prov", "Sf", "Broker", "Trust", "Orch")
    foreach ($serviceType in $serviceTypes)
    {
        RegisterServiceInstances $serviceType
    }

    Write-Host "ServiceInstances have been upgraded"
}

Function UpgradeAdminConfiguration()
{
    Import-ConfigFeatureTable  -Path "C:\Program Files\Citrix\XenDesktopPoshSdk\Module\Citrix.XenDesktop.Admin.V1\Citrix.XenDesktop.Admin\FeatureTable.xml"
    Import-AdminRoleConfiguration -Path "C:\Program Files\Citrix\XenDesktopPoshSdk\Module\Citrix.XenDesktop.Admin.V1\Citrix.XenDesktop.Admin\StudioRoleConfig\RoleConfigSigned.xml"
    Import-AdminRoleConfiguration -Path "C:\Program Files\Citrix\XenDesktopPoshSdk\Module\Citrix.XenDesktop.Admin.V1\Citrix.XenDesktop.Admin\StudioRoleConfig\DirectorRoleConfigSigned.xml"

    $site=Set-ConfigSite -ProductVersion "7.41"

    $serviceTypes=@("Config", "Admin", "Log", "Acct", "Analytics", "AppLib", "EnvTest", "Hyp", "Prov", "Sf", "Broker", "Trust", "Orch")
    foreach ($serviceType in $serviceTypes)
    {
        $cmdlet="Reset-{0}EnabledFeatureList" -f $serviceType  
        Invoke-Expression $cmdlet
    }

    New-AdminAdministrator  -IsHidden $True -Name "S-1-5-20" -ErrorAction "SilentlyContinue"
    Add-AdminRight -Administrator "S-1-5-20" -Role "fd793b26-1c19-407f-92ec-1c4177fed7b4" -Scope "00000000-0000-0000-0000-000000000000"
    Start-OrchRestApi

    Write-Host "AdminConfiguration has been upgraded"
}

UpgradeServiceInstances
UpgradeAdminConfiguration
```
{% endraw %}
