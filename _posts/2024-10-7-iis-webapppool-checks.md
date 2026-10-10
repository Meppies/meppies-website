---
layout: post
title: Check IIS Web App Pool state in an Ansible Library
Date: 2024-10-07
categories: [ Ansible, Powershell, Citrix, CVAD]
thumbnail: "assets/images/database_upgrade.png"
description: This library assists in assessing the health status of a IIS Web App Pools.
---

### How to use

This library assists in assessing the health status of a IIS Web App Pools and offering one configurable options:

- iis_webapppool::  This parameter specifies the IIS Web App Pool you like to check.
{% raw %}
```yaml
    - name: Check if iis web apppools are running on all storefront servers
      win_iis_webapppool_check:
        iis_webapppool: "CitrixADIdentityService"

```
{% endraw %}


### Library code

To integrate the IIS web apppool check into Ansible, create a YAML file named win_iis_webapppool_check.ps1 to store it in the library folder
{% raw %}
```yaml
#!powershell

#Requires -Module Ansible.ModuleUtils.Legacy
#Requires -Module Ansible.ModuleUtils.Backup


$params = Parse-Args $args -supports_check_mode $true
$iisWEBAPPPOOL = Get-AnsibleParam $params "iis_webapppool" -type "str" -FailIfEmpty $true


$result = @{
    changed = $false
}

try { 

    $status = Get-WebAPPPOOLState $iisWEBAPPPOOL
    $status = $status.value
    $status   

    $searchString = 'Stopped'
    if ($status -contains $searchString){
        Fail-Json $result "webapppool $iisWEBAPPPOOL is not started "
    } else {
        Exit-Json $result "webapppool $iisWEBAPPPOOL started"
    }  
}
catch {
    Fail-Json $result "unable to check WEBAPPPOOL $iisWEBAPPPOOL"
}
}
```
{% endraw %}