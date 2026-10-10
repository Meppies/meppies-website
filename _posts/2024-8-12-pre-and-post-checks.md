---
layout: post
title: Pre and post checks for CVAD
Date: 2024-08-12
categories: [ Ansible, Powershell, Citrix, CVAD]
thumbnail: "assets/images/database_upgrade.png"
description: To ensure server readiness for checks or installations, it's essential to verify its health status. To facilitate this, I've implemented several checks to assess server health. 
---

## My checks

The following code outlines the specific checks implemented to assess server health.

### Pre checks Delivery controller


{% raw %}
```yaml
- name: verify registry hklm_software_citrix
  win_reg_stat:
    path: HKLM:\SOFTWARE\Citrix
  register: hklm_software_citrix

- name: Pre Delivery Controller Check
  block:

    - name: Check if all the Citrix Delivery Controllers State is Active
      citrix_deliverycontroller_check:
      delegate_to: "{{ item }}"
      loop: "{{ groups['CitrixDeliveryController'] }}"
    
    - name: Check if win services are running on all Delivery Controller servers
      win_service_check:
        win_service: "{{ item.0 }}"
      delegate_to: "{{ item.1 }}"
      loop: "{{ checks.citrix.delivery_controller.win_services | product(groups['CitrixDeliveryController'])|list }}"

  when:
    - inventory_hostname in groups["CitrixDeliveryController"]
    - hklm_software_citrix.exists == true
```
{% endraw %}


### Post checks Delivery controller


{% raw %}
```yaml
- name: verify registry hklm_software_citrix
  win_reg_stat:
    path: HKLM:\SOFTWARE\Citrix
  register: hklm_software_citrix

- name: Post Delivery Controller Check
  block:

    - name: Check if this Citrix Delivery Controller State is Active
      citrix_deliverycontroller_check:
      register: ddc_check_state
      until: ddc_check_state.failed == false
      retries: 20
      delay: 15

    - name: Check if win services are running on this Delivery Controller server
      win_service_check:
        win_service: "{{ item }}"
      loop: "{{ checks.citrix.delivery_controller.win_services }}"
      retries: 10
      delay: 60
      
  when:
    - inventory_hostname in groups["CitrixDeliveryController"]
    - hklm_software_citrix.exists == true
```
{% endraw %}


### Pre checks Director


{% raw %}
```yaml
- name: verify registry hklm_software_citrix
  win_reg_stat:
    path: HKLM:\SOFTWARE\Citrix
  register: hklm_software_citrix

- name: Pre Director check and remove from load
  block:

    - name: Check if win services are running on all Director servers
      win_service_check:
        win_service: "{{ item.0 }}"
      delegate_to: "{{ item.1 }}"
      loop: "{{ checks.citrix.director.win_services | product(groups['CitrixDirector'])|list }}"

    - name: Check if iis web apppools are running on all Director servers
      win_iis_webapppool_check:
        iis_webapppool: "{{ item.0 }}"
      delegate_to: "{{ item.1 }}"
      loop: "{{ checks.citrix.director.iis_webapppools | product(groups['CitrixDirector'])|list }}"

  when:
    - inventory_hostname in groups["CitrixDirector"]
    - hklm_software_citrix.exists == true
```
{% endraw %}


### Post checks Director


{% raw %}
```yaml
- name: verify registry hklm_software_citrix
  win_reg_stat:
    path: HKLM:\SOFTWARE\Citrix
  register: hklm_software_citrix

- name: Post Director check and add to load
  block:

    - name: Check if win services are running on this Director server
      win_service_check:
        win_service: "{{ item }}"
      loop: "{{ checks.citrix.director.win_services }}"
      retries: 10
      delay: 60

    - name: Check if iis web apppools are running on this Director server
      win_iis_webapppool_check:
        iis_webapppool: "{{ item }}"
      loop: "{{ checks.citrix.director.iis_webapppools }}"

  when:
    - inventory_hostname in groups["CitrixDirector"]
    - hklm_software_citrix.exists == true
```
{% endraw %}


### Pre checks Storefront


{% raw %}
```yaml
- name: verify registry hklm_software_citrix
  win_reg_stat:
    path: HKLM:\SOFTWARE\Citrix
  register: hklm_software_citrix

- name: Pre Storefront check and remove from load
  block:

    # temp fix until SOLLAR for Storefront is ok  
    - name: Add members to Administrator group
      win_group_membership:
        name: Administrators
        members:
          - "NT SERVICE\\CitrixClusterService"
          - "NT SERVICE\\CitrixConfigurationReplication"
        state: present
      delegate_to: "{{ item }}"
      loop: "{{ groups['CitrixStoreFront'] }}"

    - name: Start CitrixConfigurationReplication - TEMP FIX
      ansible.windows.win_service:
        name: CitrixConfigurationReplication
        start_mode: auto
        state: started
      delegate_to: "{{ item }}"
      loop: "{{ groups['CitrixStoreFront'] }}"
    
    - name: Restart CitrixConfigurationReplication
      ansible.windows.win_service:
        name: CitrixConfigurationReplication
        state: restarted
      delegate_to: "{{ item }}"
      loop: "{{ groups['CitrixStoreFront'] }}"

    - name: Check if win services are running on all storefront servers
      win_service_check:
        win_service: "{{ item.0 }}"
      delegate_to: "{{ item.1 }}"
      loop: "{{ checks.citrix.storefront.win_services | product(groups['CitrixStoreFront'])|list }}"

    - name: Check if iis web apppools are running on all storefront servers
      win_iis_webapppool_check:
        iis_webapppool: "{{ item.0 }}"
      delegate_to: "{{ item.1 }}"
      loop: "{{ checks.citrix.storefront.iis_webapppools | product(groups['CitrixStoreFront'])|list }}"

  when:
    - inventory_hostname in groups["CitrixStoreFront"]
    - hklm_software_citrix.exists == true
```
{% endraw %}


### Post checks Storefront


{% raw %}
```yaml
- name: verify registry hklm_software_citrix
  win_reg_stat:
    path: HKLM:\SOFTWARE\Citrix
  register: hklm_software_citrix

- name: Post Storefront check and add to load
  block:

    - name: Check if win services are running on this Storefront server
      win_service_check:
        win_service: "{{ item }}"
      loop: "{{ checks.citrix.storefront.win_services }}"
      retries: 10
      delay: 60

    - name: Check if iis web apppools are running on this Storefront server
      win_iis_webapppool_check:
        iis_webapppool: "{{ item }}"
      loop: "{{ checks.citrix.storefront.iis_webapppools }}"

  when:
    - inventory_hostname in groups["CitrixStoreFront"]
    - hklm_software_citrix.exists == true
```
{% endraw %}


### Pre checks License server


{% raw %}
```yaml
- name: verify registry hklm_software_citrix
  win_reg_stat:
    path: HKLM:\SOFTWARE\Citrix
  register: hklm_software_citrix

- name: Pre Citrix License server Check
  block:

    - name: Check if win services are running on all Citrix License servers
      win_service_check:
        win_service: "{{ item.0 }}"
      delegate_to: "{{ item.1 }}"
      loop: "{{ checks.citrix.license.win_services | product(groups['CitrixLicenseServer'])|list }}"

  when:
    - inventory_hostname in groups["CitrixLicenseServer"]
    - hklm_software_citrix.exists == true
```
{% endraw %}


### Post checks License server


{% raw %}
```yaml
- name: verify registry hklm_software_citrix
  win_reg_stat:
    path: HKLM:\SOFTWARE\Citrix
  register: hklm_software_citrix

- name: Post Citrix License server Check
  block:

    - name: Check if win services are running on all Citrix License servers
      win_service_check:
        win_service: "{{ item }}"
      loop: "{{ checks.citrix.license.win_services }}"
      retries: 10
      delay: 60

  when:
    - inventory_hostname in groups["CitrixLicenseServer"]
    - hklm_software_citrix.exists == true
```
{% endraw %}


### Pre checks SQL Always ON


{% raw %}
```yaml
- name: Pre SQL Check and remove from load
  block:

    - name: Check if the SQL cluster Service is running
      win_shell: |
        $status = get-service ClusSvc |select $status
        $status = $status.status
        $status
      delegate_to: "{{ item }}"
      register: sql_cluster_svc_check
      failed_when: sql_cluster_svc_check.stdout != "Running\r\n"
      loop: "{{ groups['SQLServer'] }}"

    - name: Obtain information from registry for ServerType
      win_reg_stat:
        path: HKLM:\Software\#####
        name: ServerType
      register: server_type

    - name: define environment to availability group
      set_fact:
        availability_group:
          D: "{{ sql.availability_group.dev }}"
          T: "{{ sql.availability_group.test }}"
          A: "{{ sql.availability_group.non_prod }}"
          P: "{{ sql.availability_group.prod }}"

    - name: get PrimaryReplicaServerName
      win_shell: |
        Import-Module sqlserver
        $primaryReplicaServerName = (Get-ChildItem SQLSERVER:\sql\{{ inventory_hostname_short }}\default\AvailabilityGroups).PrimaryReplicaServerName
        return $primaryReplicaServerName
      register: output_primary_replica

    - name: set fact primary_replica
      set_fact:
        primary_replica: "{{ output_primary_replica.stdout_lines [0] + '.((FQDN Domain))' }}"

    - name: set fact primary_replica_short
      set_fact:
        primary_replica_short:  "{{ output_primary_replica.stdout_lines [0] }}"

    - name: Test-SqlAvailabilityGroup
      win_shell: |
        Import-Module sqlserver
        $healthState = Test-SqlAvailabilityGroup -Path SQLSERVER:\Sql\{{ primary_replica_short }}\\Default\AvailabilityGroups\{{ availability_group[server_type.value] }}
        $healthState = $healthState.HealthState
        $healthState
      register: sql_health_state_check
      failed_when: sql_health_state_check.stdout != "Healthy\r\n"
      delegate_to: "{{ primary_replica }}"

    - name: set fact delegate_to_hosts
      set_fact:
        delegate_to_hosts: "{{ groups['SQLServer']|difference(inventory_hostname) | random }}"

    - name: set fact delegate_to_hosts_short
      set_fact:
        delegate_to_hosts_short: "{{ delegate_to_hosts | replace ('((FQDN Domain))','')}}"

    - name: Switch-SqlAvailabilityGroup to other node if this server is primary
      win_shell: |
        Import-Module sqlserver
        Switch-SqlAvailabilityGroup -Path SQLSERVER:\Sql\{{ delegate_to_hosts_short }}\Default\AvailabilityGroups\{{ availability_group[server_type.value] }}
      delegate_to: "{{ delegate_to_hosts }}"
      when: primary_replica == inventory_hostname

    - name: Change Failover mode to Manual for patching
      win_shell: | 
        Import-Module sqlserver
        $node1FQDN = "{{ groups['SQLServer'][0] }}"
        $node1 = $node1FQDN.split(".")[0]
        $node2FQDN = "{{ groups['SQLServer'][1] }}"
        $node2 = $node2FQDN.split(".")[0]
        Set-SqlAvailabilityReplica -AvailabilityMode "SynchronousCommit" -FailoverMode "Manual" -Path "SQLSERVER:\Sql\{{ delegate_to_hosts_short }}\Default\AvailabilityGroups\{{ availability_group[server_type.value] }}\AvailabilityReplicas\$node1"
        Set-SqlAvailabilityReplica -AvailabilityMode "SynchronousCommit" -FailoverMode "Manual" -Path "SQLSERVER:\Sql\{{ delegate_to_hosts_short }}\Default\AvailabilityGroups\{{ availability_group[server_type.value] }}\AvailabilityReplicas\$node2"
      delegate_to: "{{ delegate_to_hosts }}"

  when: inventory_hostname in groups["SQLServer"]
```
{% endraw %}


### Post checks SQL Always ON


{% raw %}
```yaml
- name: Post SQL Check and add to load
  block:
    - name: Check if the SQL cluster Service is running
      win_shell: |
        $status = get-service ClusSvc |select $status
        $status = $status.status
        $status
      delegate_to: "{{ item }}"
      register: sql_cluster_svc_check
      failed_when: sql_cluster_svc_check.stdout != "Running\r\n"
      loop: "{{ groups['SQLServer'] }}"

    - name: Obtain information from registry for ServerType
      win_reg_stat:
        path: HKLM:\Software\####
        name: ServerType
      register: server_type

    - name: define environment to availability group
      set_fact:
        availability_group:
          D: "{{ sql.availability_group.dev }}"
          T: "{{ sql.availability_group.test }}"
          A: "{{ sql.availability_group.non_prod }}"
          P: "{{ sql.availability_group.prod }}"

    - name: get PrimaryReplicaServerName
      win_shell: |
        Import-Module sqlserver
        $primaryReplicaServerName = (Get-ChildItem SQLSERVER:\sql\{{ inventory_hostname_short }}\default\AvailabilityGroups).PrimaryReplicaServerName
        return $primaryReplicaServerName
      register: output_primary_replica

    - name: set fact primary_replica
      set_fact:
        primary_replica: "{{ output_primary_replica.stdout_lines [0] + '((FQDN Domain))' }}"

    - name: set fact primary_replica_short
      set_fact:
        primary_replica_short:  "{{ output_primary_replica.stdout_lines [0] }}"

    - name: Test-SqlAvailabilityGroup
      win_shell: |
        Import-Module sqlserver
        $healthState = Test-SqlAvailabilityGroup -Path SQLSERVER:\Sql\{{ primary_replica_short }}\\Default\AvailabilityGroups\{{ availability_group[server_type.value] }}
        $healthState = $healthState.HealthState
        $healthState
      register: sql_health_state_check
      delegate_to: "{{ primary_replica }}"
      until: sql_health_state_check.stdout == "Healthy\r\n"
      retries: 30
      delay: 60
      
    - name: Change Failover mode to Automatic after patching
      win_shell: | 
        Import-Module sqlserver
        $node1FQDN = "{{ groups['SQLServer'][0] }}"
        $node1 = $node1FQDN.split(".")[0]
        $node2FQDN = "{{ groups['SQLServer'][1] }}"
        $node2 = $node2FQDN.split(".")[0]
        Set-SqlAvailabilityReplica -AvailabilityMode "SynchronousCommit" -FailoverMode "Automatic" -Path "SQLSERVER:\Sql\{{ primary_replica_short }}\Default\AvailabilityGroups\{{ availability_group[server_type.value] }}\AvailabilityReplicas\$node1"
        Set-SqlAvailabilityReplica -AvailabilityMode "SynchronousCommit" -FailoverMode "Automatic" -Path "SQLSERVER:\Sql\{{ primary_replica_short }}\Default\AvailabilityGroups\{{ availability_group[server_type.value] }}\AvailabilityReplicas\$node2"
      delegate_to: "{{ primary_replica }}"

  when: inventory_hostname in groups["SQLServer"]
```
{% endraw %}