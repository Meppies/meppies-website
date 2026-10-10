---
layout: post
title: Ansible Citrix License server admins library
Date: 2024-03-04
categories: [Citrix, CVAD, Ansible, Powershell, License server]
thumbnail: "assets/images/database_upgrade.png"
description: To streamline Citrix license server administration, I developed a reusable Ansible library that simplifies common tasks and enhances automation capabilities.
---

### How to use

This lightweight Ansible library offers three configurable options:

- license_server: This mandatory parameter specifies the fully qualified domain name (FQDN) of the Citrix License Server.
- lic_admins: This optional parameter defines a list of users or groups who will be granted full administrative privileges on the license server.
- lic_readonlys: This optional parameter defines a list of users or groups who will be granted read-only access to the license server.
{% raw %}
```yaml
- name: Configure Citrix License Config
  citrix_license_config:
    license_server: "{{ inventory_hostname }}"
    lic_admins:
      - "{{ admin name or group }}" 
      - "{{ admin name or group }}" 
    lic_readonlys:
      - "{{ admin name or group }}" 

```
{% endraw %}


### Library code

To integrate the Citrix licensing configuration into Ansible, create a YAML file named citrix_license_config.yaml to store it in the library folder
{% raw %}
```yaml
#!powershell

#Requires -Module Ansible.ModuleUtils.Legacy
#Requires -Module Ansible.ModuleUtils.Backup

$params = Parse-Args $args -supports_check_mode $true

$LicenseServer = Get-AnsibleParam $params "license_server" -type "str" -FailIfEmpty $true
$LicAdmins = Get-AnsibleParam $params "lic_admins" -type "list"
$Licreadonlys = Get-AnsibleParam $params "lic_readonlys" -type "list"

$updatepermissions = $false
$compareadmin = ""
$comparereadonly = ""
$certhash = (Get-LicCertificate -AdminAddress "https://$($LicenseServer):8083/").certhash
$result = @{
    changed = $false
}

## Get the admins of the license server
try {
    $AdminResult = get-LicAdministrator -AdminAddress "https://$($LicenseServer):8083/" -certhash $certhash
}
catch {
    Fail-Json $result "failed to retrieve admins from $LicenseServer error: $_"
}

## Remove buildin Administrator from the list
try {
    $AdminResult = $AdminResult | Where-Object { $_.account -ne "BUILTIN\ADMINISTRATORS" }
}
catch {
    Fail-Json $result "failed to remove the build in admin from the admin list. $LicenseServer error: $_"
}

## Compare License admins with the input admins
try {
    $AdminResultadmin = $AdminResult | Where-Object { $_.Permissions -eq "Full" }

    if ($AdminResultadmin -ne $null) {
        $compareadmin = Compare-Object -ReferenceObject $LicAdmins -DifferenceObject $AdminResultadmin.account
    }
    else {
        $updatepermissions = $true
    }

    if ($compareadmin -ne $null) {
        $updatepermissions = $true
    }
}
catch {
    Fail-Json $result "failed to compare the admins with the license server $LicenseServer error: $_"
}

## Compare License readonly with the input readonly
try {
    $AdminResultreadonly = $AdminResult | Where-Object { $_.Permissions -eq "readOnly" }
    
    if ($AdminResultreadonly -ne $null) {
        $comparereadonly = Compare-Object -ReferenceObject $Licreadonlys -DifferenceObject $AdminResultreadonly.account
    }
    else {
        $updatepermissions = $true
    }

    if ($comparereadonly -ne $null) {
        $updatepermissions = $true
    }
}
catch {
    Fail-Json $result "failed to compare the readonly with the license server $LicenseServer error: $_"
}

## remove all the admins from the server
try {
    if ($updatepermissions -eq $true) {
        foreach ($admin in $AdminResult) {
            Remove-LicAdministrator -AdminAddress "https://$($LicenseServer):8083/" -Account $admin.Account -certhash $certhash
        }
    }
}
catch {
    Fail-Json $result "failed to remove the admins from the server $LicenseServer error: $_"
}

## Add admins to the License server
try {
    if ($updatepermissions -eq $true) {
        foreach ($newadmin in $LicAdmins) {
            try {
                New-LicAdministrator -AdminAddress "https://$($LicenseServer):8083/" -Account $newadmin -Group -certhash $certhash
            }
            ## Remote we get the error ActiveDirectoryAccountResolutionFailed user is added so we can skip this error
            Catch [System.InvalidOperationException] {
                "just proceed: $error"
            }
        }
        $result.changed = $true
    }
}
## Remote we get the error ActiveDirectoryAccountResolutionFailed user is added so we can skip this error
Catch [System.InvalidOperationException] {
    "just proceed: $error"
}
catch {
    Fail-Json $result "failed to add admin $newadmin to $LicenseServer error: $_"
}

## Add readonlys to the License server
try {
    if ($updatepermissions -eq $true) {
        foreach ($newreadonly in $Licreadonlys) {
            try {
                New-LicAdministrator -AdminAddress "https://$($LicenseServer):8083/" -Account $newreadonly -ReadOnly -Group -certhash $certhash
            }
            ## Remote we get the error ActiveDirectoryAccountResolutionFailed user is added so we can skip this error
            Catch [System.InvalidOperationException] {
                "just proceed: $error"
            }
        }
        $result.changed = $true
    }
}
catch {
    Fail-Json $result "failed to add readonly $newreadonly to $LicenseServer error: $_"
}

Exit-Json $result

```
{% endraw %}