---
layout: post
title: Pipeline schedule for Citrix desired state configuration
Date: 2024-04-08
categories: [Citrix, CVAD, Ansible, Azure Devops, Pipeline, Desired state]
thumbnail: "assets/images/database_upgrade.png"
description: To ensure consistent and predictable Citrix configurations, frequent pipeline execution is crucial. This helps maintain the desired state of the environment. However, due to Azure DevOps scheduling limitations, a custom script was developed to circumvent these restrictions and enable more flexible automation.
---

### Why?

At a recent client engagement, we implemented a single pipeline to handle various tasks with different parameterizations. However, Azure DevOps's scheduling limitations prevent direct parameterization. To overcome this constraint, we developed a solution that enables the scheduling of pipelines with specific parameters, offering greater control and flexibility.
<center><img src="/assets/img/azure-devops-pipeline-menu.png" alt="" width="40%" height="auto" style="border-radius: 20px" /></center>


### Steps

#### Create parameters

To maintain flexibility, a parameter named 'Run the Pipeline' has been introduced. This parameter allows the pipeline to be executed manually, independent of the scheduled runs.
{% raw %}
```yaml
parameters:
- name: update_run_boolean
  displayName: Run the pipeline
  type: boolean
  default: false
```
{% endraw %}

#### Create Cron schedule

A cron schedule will be implemented to execute the pipeline hourly between 01:00 and 16:00 UTC for all DTAP environments.
{% raw %}
```yaml
schedules:
- cron: '0 1-16 * * *'
  displayName: Run every hour between 1-16
  branches:
    include:
    - develop
    - test
    - accept
    - master
  always: true
```
{% endraw %}

#### New Stage

Define a new stage in the pipeline, consisting of multiple jobs.
{% raw %}
```yaml
- stage: schedule_check
  displayName: Check schedule
  jobs: 
```
{% endraw %}

#### Get the date and time

A job has been created to determine the current day and hour, storing the values in a variable for subsequent job execution. To prevent unnecessary execution when the 'Run the Pipeline' parameter is set to true, a conditional check has been implemented.
{% raw %}
```yaml
- job: get_week_and_time
condition: |
    ${{ eq(parameters.update_run_boolean, false) }}
displayName: Check the week of the Month and Time
steps:
- powershell: |
        #get the day and time of today
        $d = Get-Date
        
        #Check if it's a Monday and the first week of the month
        if ($d.DayOfWeek -eq 'Monday' -and $d.Day -in 0..7) {
            Write-Host "##vso[task.setvariable variable=first_monday;isOutput=true]$($true)"
            }
        else{
            Write-Host "##vso[task.setvariable variable=first_monday;isOutput=true]$($false)"
            }
        
        #Check if it's a Monday and the second week of the month
        if ($d.DayOfWeek -eq 'Monday' -and $d.Day -in 8..14) {
            Write-Host "##vso[task.setvariable variable=second_monday;isOutput=true]$($true)"
            }
        else{
            Write-Host "##vso[task.setvariable variable=second_monday;isOutput=true]$($false)"
            }
        
        #Check if it's a Monday and the third week of the month
        if ($d.DayOfWeek -eq 'Monday' -and $d.Day -in 15..21) {
            Write-Host "##vso[task.setvariable variable=third_monday;isOutput=true]$($true)"
            }
        else{
            Write-Host "##vso[task.setvariable variable=third_monday;isOutput=true]$($false)"
            }
        
        #Set the the hours of the day in a variable
        Write-Host "##vso[task.setvariable variable=time_check;isOutput=true]$($d.Hour)"
    name: get_week_time
    displayName: Get the week of the Month and time

```
{% endraw %}

#### Check empty variables

To prevent pipeline failures caused by empty variables, an additional check has been implemented. This check ensures that the pipeline continues execution when the 'Run the Pipeline' parameter is selected.
{% raw %}
```yaml
- job: check_if_values_empty
condition: |
    in(dependencies.get_week_and_time.result, 'Succeeded', 'SucceededWithIssues', 'Skipped', 'Failed')
dependsOn:
- get_week_and_time
displayName: Check if values are empty
variables:
    firstweek_check: $[Dependencies.get_week_and_time.outputs['get_week_time.first_monday']]
    secondweek_check: $[Dependencies.get_week_and_time.outputs['get_week_time.second_monday']]
    thirdweek_check: $[Dependencies.get_week_and_time.outputs['get_week_time.third_monday']]
    time_check_value: $[Dependencies.get_week_and_time.outputs['get_week_time.time_check']]
steps:
- powershell: |
        $firstweek = "$(firstweek_check)"
        $secondweek = "$(secondweek_check)"
        $thirdweek = "$(thirdweek_check)"
        $time_check = "$(time_check_value)"

        if($firstweek -eq "true"){
            Write-Host "##vso[task.setvariable variable=first_monday;isOutput=true]$($true)"
        }
        else{
            Write-Host "##vso[task.setvariable variable=first_monday;isOutput=true]$($false)"
        }
        if($secondweek -eq "true"){
            Write-Host "##vso[task.setvariable variable=second_monday;isOutput=true]$($true)"
        }
        else{
            Write-Host "##vso[task.setvariable variable=second_monday;isOutput=true]$($false)"
        }
        if($thirdweek -eq "true"){
            Write-Host "##vso[task.setvariable variable=third_monday;isOutput=true]$($true)"
        }
        else{
            Write-Host "##vso[task.setvariable variable=third_monday;isOutput=true]$($false)"
        }
        if($time_check){
            Write-Host "##vso[task.setvariable variable=time_check;isOutput=true]$time_check"
        }
        else{
            Write-Host "##vso[task.setvariable variable=time_check;isOutput=true]'0'"
        }
    name: update_value
    displayName: Update Value if empty
```
{% endraw %}

#### Check the variables

The final job verifies the accuracy of the time-related variables, ensuring they align with the specified day and hour or the 'Run the Pipeline' parameter. The results are stored in a variable for subsequent stages. This job is made dependent on the 'Check empty variables' job to guarantee sequential execution and prevent concurrent runs. Additionally, the pipeline will fail if the 'Check empty variables' job does not complete successfully, ensuring that all necessary variables are properly set.

{% raw %}
```yaml
- job: check_if_meets_criteria
condition: |
    in(dependencies.check_if_values_empty.result, 'Succeeded')
dependsOn:
- check_if_values_empty
displayName: Check if criteria set to run schedule
variables:
    firstweek: $[Dependencies.check_if_values_empty.outputs['update_value.first_monday']]
    secondweek: $[Dependencies.check_if_values_empty.outputs['update_value.second_monday']]
    thirdweek: $[Dependencies.check_if_values_empty.outputs['update_value.third_monday']]
    time_check: $[Dependencies.check_if_values_empty.outputs['update_value.time_check']]
    branchcheck: $[replace(variables['Build.SourceBranch'], 'refs/heads/', '')]
    updateConfigurationBoolean: ${{ parameters.update_configuration_boolean }}
    updateCertificateBoolean: ${{ parameters.update_certificate_boolean }}
    updateSoftwareBoolean: ${{ parameters.update_software_boolean }}
    updaterunboolean: ${{ parameters.update_run_boolean }}
steps:
- powershell: |
        $firstweek_ps = "$(firstweek)"
        $secondweek_ps = "$(secondweek)"
        $thirdweek_ps = "$(thirdweek)"
        $time_check_ps = "$(time_check)"
        $branchcheck_ps = "$(branchcheck)"
        $updateConfigurationBoolean_ps = "$(updateConfigurationBoolean)"
        $updateCertificateBoolean_ps = "$(updateCertificateBoolean)"
        $updateSoftwareBoolean_ps = "$(updateSoftwareBoolean)"
        $updaterunboolean_ps = "$(updaterunboolean)"

        #check if the variable updaterunboolean is true 
        if ($updaterunboolean_ps -eq 'false'){
            
            #check if the branch is develop and the time is 4 hours ETC
            if($branchcheck_ps -eq "develop" -and $time_check_ps -eq "4"){
            Write-Host "##vso[task.setvariable variable=check_updateConfiguration;isOutput=true]$($true)"
            Write-Host "##vso[task.setvariable variable=check_updateCertificate;isOutput=true]$($false)"
            Write-Host "##vso[task.setvariable variable=check_updateSoftware;isOutput=true]$($false)"
            Write-Host "##vso[task.setvariable variable=check_updaterun;isOutput=true]$($true)"
            }
            
            #check if the branch is test and the time is 6 hours ETC
            if($branchcheck_ps -eq "test" -and $time_check_ps -eq "6"){
            Write-Host "##vso[task.setvariable variable=check_updateConfiguration;isOutput=true]$($true)"
            Write-Host "##vso[task.setvariable variable=check_updateCertificate;isOutput=true]$($false)"
            Write-Host "##vso[task.setvariable variable=check_updateSoftware;isOutput=true]$($false)"
            Write-Host "##vso[task.setvariable variable=check_updaterun;isOutput=true]$($true)"
            }

            #check if the branch is non-prod and the time is 8 hours ETC
            if($branchcheck_ps -eq "accept" -and $time_check_ps -eq "8"){
            Write-Host "##vso[task.setvariable variable=check_updateConfiguration;isOutput=true]$($true)"
            Write-Host "##vso[task.setvariable variable=check_updateCertificate;isOutput=true]$($false)"
            Write-Host "##vso[task.setvariable variable=check_updateSoftware;isOutput=true]$($false)"
            Write-Host "##vso[task.setvariable variable=check_updaterun;isOutput=true]$($true)"
            }

            #check if the branch is prod and the time is 11 hours ETC
            if($branchcheck_ps -eq "master" -and $time_check_ps -eq "11"){
            Write-Host "##vso[task.setvariable variable=check_updateConfiguration;isOutput=true]$($true)"
            Write-Host "##vso[task.setvariable variable=check_updateCertificate;isOutput=true]$($false)"
            Write-Host "##vso[task.setvariable variable=check_updateSoftware;isOutput=true]$($false)"
            Write-Host "##vso[task.setvariable variable=check_updaterun;isOutput=true]$($true)"
            }

            #check if the branch is develop, the time is 5 hours ETC and it the first week of the month
            if($branchcheck_ps -eq "develop" -and $time_check_ps -eq "5" -and $firstweek_ps -eq 'true'){
            Write-Host "##vso[task.setvariable variable=check_updateConfiguration;isOutput=true]$($false)"
            Write-Host "##vso[task.setvariable variable=check_updateCertificate;isOutput=true]$($true)"
            Write-Host "##vso[task.setvariable variable=check_updateSoftware;isOutput=true]$($false)"
            Write-Host "##vso[task.setvariable variable=check_updaterun;isOutput=true]$($true)"
            }
            
            #check if the branch is test, the time is 7 hours ETC and it the first week of the month
            if($branchcheck_ps -eq "test" -and $time_check_ps -eq "7" -and $firstweek_ps -eq 'true'){
            Write-Host "##vso[task.setvariable variable=check_updateConfiguration;isOutput=true]$($false)"
            Write-Host "##vso[task.setvariable variable=check_updateCertificate;isOutput=true]$($true)"
            Write-Host "##vso[task.setvariable variable=check_updateSoftware;isOutput=true]$($false)"
            Write-Host "##vso[task.setvariable variable=check_updaterun;isOutput=true]$($true)"
            }

            #check if the branch is non-prod, the time is 9 hours ETC and it the second week of the month
            if($branchcheck_ps -eq "accept" -and $time_check_ps -eq "9" -and $secondweek_ps -eq 'true'){
            Write-Host "##vso[task.setvariable variable=check_updateConfiguration;isOutput=true]$($false)"
            Write-Host "##vso[task.setvariable variable=check_updateCertificate;isOutput=true]$($true)"
            Write-Host "##vso[task.setvariable variable=check_updateSoftware;isOutput=true]$($false)"
            Write-Host "##vso[task.setvariable variable=check_updaterun;isOutput=true]$($true)"
            }

            #check if the branch is prod, the time is 5 hours ETC and it the third week of the month
            if($branchcheck_ps -eq "master" -and $time_check_ps -eq "12" -and $thirdweek_ps -eq 'true'){
            Write-Host "##vso[task.setvariable variable=check_updateConfiguration;isOutput=true]$($false)"
            Write-Host "##vso[task.setvariable variable=check_updateCertificate;isOutput=true]$($true)"
            Write-Host "##vso[task.setvariable variable=check_updateSoftware;isOutput=true]$($false)"
            Write-Host "##vso[task.setvariable variable=check_updaterun;isOutput=true]$($true)"
            }
            }
        
        # If updaterunboolean variable is true take over the variable from the pipeline it self
        else {
            Write-Host "##vso[task.setvariable variable=check_updaterun;isOutput=true]$($true)"

            if($updateConfigurationBoolean_ps -eq "true"){
            Write-Host "Set Configuration to true"
            Write-Host "##vso[task.setvariable variable=check_updateConfiguration;isOutput=true]$($true)"
            }
            else {
            Write-Host "##vso[task.setvariable variable=check_updateConfiguration;isOutput=true]$($false)"
            Write-Host "Set Configuration to false"
            }

            if($updateCertificateBoolean_ps -eq "true"){
            Write-Host "##vso[task.setvariable variable=check_updateCertificate;isOutput=true]$($true)"
            }
            else {
            Write-Host "##vso[task.setvariable variable=check_updateCertificate;isOutput=true]$($false)"
            }

            if($updateSoftwareBoolean_ps -eq "true"){
            Write-Host "##vso[task.setvariable variable=check_updateSoftware;isOutput=true]$($true)"
            Write-Host "##vso[task.setvariable variable=check_updateConfiguration;isOutput=true]$($true)"
            }
            else {
            Write-Host "##vso[task.setvariable variable=check_updateSoftware;isOutput=true]$($false)"
            }
        }
    name: check_if_need_to_schedule
    displayName: Check if schedule need to run
```
{% endraw %}

#### Conditions and variables on the stages

In all subsequent stages after the 'schedule_check' stage, you can now access and utilize the 'updaterun' check and the branch information. By referencing the variable within the stage, other jobs can leverage these values. It's crucial to establish dependencies between stages, ensuring that subsequent stages only execute after the successful completion of the 'schedule_check' stage to maintain proper execution order and prevent potential conflicts.
{% raw %}
```yaml
condition: |
    and(
    eq(Dependencies.schedule_check.outputs['check_if_meets_criteria.check_if_need_to_schedule.check_updaterun'], 'true'),
        or(
        contains(variables['build.sourceBranch'], 'refs/heads/master'), 
        contains(variables['build.sourceBranch'], 'refs/heads/accept'), 
        contains(variables['build.sourceBranch'], 'refs/heads/test'), 
        contains(variables['build.sourceBranch'], 'refs/heads/develop')
        )
    )
variables:
    updateConfigurationBoolean: $[stageDependencies.schedule_check.check_if_meets_criteria.outputs['check_if_need_to_schedule.check_updateConfiguration']]
    updateCertificateBoolean: $[stageDependencies.schedule_check.check_if_meets_criteria.outputs['check_if_need_to_schedule.check_updateCertificate']]
    updateSoftwareBoolean: $[stageDependencies.schedule_check.check_if_meets_criteria.outputs['check_if_need_to_schedule.check_updateSoftware']]
dependsOn:
    - schedule_check
```
{% endraw %}