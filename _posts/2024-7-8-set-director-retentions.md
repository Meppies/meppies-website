---
layout: post
title: Set Director retention in an Ansible Library
Date: 2024-07-08
categories: [ Ansible, Powershell, Citrix, CVAD]
thumbnail: "assets/images/database_upgrade.png"
description: To streamline the process of modifying log data retention settings in Citrix Director, I developed a library that simplifies the configuration changes.
---

### How to use

This library enables the verification and modification of log retention settings within Citrix Director. It should be executed on a Delivery Controller and offering two configurable options:

- "setting: This parameter specifies the desired retention policy. Refer to the following options for available retention settings: [retention](https://docs.citrix.com/en-us/citrix-virtual-apps-desktops/director/data-retention.html)
- value: Specifies the number of days for which log data should be retained.
{% raw %}
```yaml
  - name: Update Citrix Director Retention
    citrix_director_retention:
      setting: GroomMachineMetricDaySummaryDataRetentionDays
      value: 365
```
{% endraw %}


### Library code

To integrate the Citrix Director retention configuration into Ansible, create a YAML file named citrix_director_retention.ps1 to store it in the library folder
{% raw %}
```yaml
#!powershell

#Requires -Module Ansible.ModuleUtils.Legacy
#Requires -Module Ansible.ModuleUtils.Backup


$params = Parse-Args $args -supports_check_mode $true

$setting = Get-AnsibleParam $params "setting" -type "str" -FailIfEmpty $true
$value = Get-AnsibleParam $params "value" -type "str" -FailIfEmpty $true

$result = @{
    changed = $false
}

try {
    $getResult = Get-MonitorConfiguration
}
catch {
    Fail-Json $result "failed to retrieve: $_"
}

try {
    if ($getResult.$setting -ne $value) {
        $parameters = @{
            $setting    =	$value
                                }
        Set-MonitorConfiguration @parameters
        $result.changed = $true
    }
}
catch {
    Fail-Json $result "failed to set: $_"
}

Exit-Json $result
```
{% endraw %}