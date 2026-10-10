---
layout: post
title: Add admin to CVAD in an Ansible Library
Date: 2024-06-03
categories: [ Ansible, Powershell, Citrix, CVAD]
thumbnail: "assets/images/database_upgrade.png"
description: To streamline the management of administrators and groups within Citrix Virtual Apps and Desktops (CVAD), I developed a library that automates the process of adding or removing users and groups.
---

### How to use

This lightweight Ansible library simplifies the management of Citrix Virtual Apps and Desktops (CVAD) administrators and groups, offering three configurable options:

- adminaddress: This mandatory parameter specifies the fully qualified domain name (FQDN) of the Citrix Delivery Controller.
- adduserorgroup: This optional parameter defines a list of users or groups who will be granded access to CVAD.
- action: This has two option 'add' or 'delete'
{% raw %}
```yaml
  - name: Add Citrix administrator group
    citrix_admin_config:
      adduserorgroup: "{{ Usetname or group}}" 
      adminaddress: "{{ Delivery controller }}:80"
      action: "add"

```
{% endraw %}


### Library code

To integrate the Citrix admin configuration into Ansible, create a YAML file named citrix_admin_config.ps1 to store it in the library folder
{% raw %}
```yaml
#!powershell

#Requires -Module Ansible.ModuleUtils.Legacy
#Requires -Module Ansible.ModuleUtils.Backup


$params = Parse-Args $args -supports_check_mode $true
$adminAddress = Get-AnsibleParam $params "adminaddress" -type "str" 
$addUserOrGroup = ("AD\" + (Get-AnsibleParam $params "adduserorgroup" -type "str"))
$action = Get-AnsibleParam $params "action" -type "str"

$result = @{
    changed = $false
}

if ($action -eq "add") {

    try { 
        Get-AdminAdministrator -AdminAddress $adminaddress -Name $addUserOrGroup -ErrorAction Stop
        Exit-Json $result "unable to create, already existing: $addUserOrGroup"
    }
    catch {
        try {
            New-AdminAdministrator  -AdminAddress $adminaddress -Enabled $True -Name $addUserOrGroup
            Add-AdminRight  -AdminAddress $adminaddress -Administrator $addUserOrGroup -Role "Full Administrator" -Scope "All"
            $result.changed = $true
            Exit-Json $result "succesfully created: $addUserOrGroup"       
        }
        catch {
            # Write-Host "unable to create new administrator $addUserOrGroup" 
            Fail-Json $result "unable to create: $addUserOrGroup" 
        }
    }
}

if ($action -eq "delete") {

    try {
        try {
            #first check if account does exist
            Get-AdminAdministrator -AdminAddress $adminaddress -Name $addUserOrGroup -ErrorAction Stop
            Remove-AdminAdministrator  -AdminAddress $adminAddress -Name $addUserOrGroup -ErrorAction Stop
            $result.changed = $True 
            Exit-Json $result "succesfully deleted: $addUserOrGroup"
        }
        catch [System.Management.Automation.ItemNotFoundException] {
            # Write-Host "specific catch"
            Exit-Json $result "object not found: $addUserOrGroup"
        }
        catch {
            # Write-Host "generic catch: $_"
            Fail-Json $result "unable to delete: $addUserOrGroup" 
        }
    }
    catch {
        #  Write-Host "unable to delete $addUserOrGroup"
        Fail-Json $result "unable to delete: $addUserOrGroup"                 
    }

}

Exit-Json $result "unknown action specified: $action"
```
{% endraw %}