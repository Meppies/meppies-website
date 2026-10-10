---
layout: post
title: Check Delivery controller state in an Ansible Library
Date: 2024-09-12
categories: [ Ansible, Powershell, Citrix, CVAD]
thumbnail: "assets/images/database_upgrade.png"
description: This library assists in assessing the health status of a Citrix Delivery Controller.
---

### How to use

This library assists in assessing the health status of a Citrix Delivery Controller and offering one configurable options:

- ad:  This parameter specifies the domain where the Delivery Controller is running in.
{% raw %}
```yaml
    - name: Check if all the Citrix Delivery Controllers State is Active
      citrix_deliverycontroller_check:
        ad: (Domain name)
      delegate_to: "{{ item }}"
      loop: "{{ groups['CitrixDeliveryController'] }}"
```
{% endraw %}


### Library code

To integrate the Citrix Delivery controller check into Ansible, create a YAML file named citrix_deliverycontroller_check.ps1 to store it in the library folder
{% raw %}
```yaml
#!powershell

#Requires -Module Ansible.ModuleUtils.Legacy
#Requires -Module Ansible.ModuleUtils.Backup

$params = Parse-Args $args -supports_check_mode $true

$ad = Get-AnsibleParam $params "ad" -type "str" -FailIfEmpty $true

$result = @{
    changed = $false
}

try { 
    $status = Get-BrokerController -MachineName "$ad\$($env:COMPUTERNAME)"
    $status = $status.State
    $status
    $searchString = 'Active'
    if ($status -contains $searchString){
        Exit-Json $result "Citrix Delivery Controller State is Active"
    } else {
        Fail-Json $result "Citrix Delivery Controller State is NOT Active"
    }  
}
catch {
    Fail-Json $result "unable to check Citrix Delivery Controller State"
}
```
{% endraw %}