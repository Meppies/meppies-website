---
layout: post
title: Pipeline send an email when stages failed
Date: 2024-05-13
categories: [ Ansible, Powershell, Azure Devops, Pipeline]
thumbnail: "assets/images/database_upgrade.png"
description: To ensure timely notification of pipeline failures in Azure DevOps, email notifications have been configured. This allows for immediate awareness of issues, even when the pipeline is scheduled to run automatically.
---

### How to use

This final stage is designed to monitor the status of preceding stages. A condition is set to trigger the stage only if any of the previous stages have failed. Once triggered, the stage can access necessary variables and send email notifications. To configure email notifications, specify the sender's email address, recipient's email address, and the SMTP server to be used for delivery.

{% raw %}
```yaml
- stage: email_failed_pipelines
  condition: |
        or(
        eq(Dependencies.schedule_check.result, 'Failed'),
        eq(Dependencies.####### 'Failed'),
        )
  displayName: Email when Pipeline Failed
  dependsOn:
        - schedule_check
  pool: CDaaSLinux
  jobs:
    - job: send_email
      displayName: send_email_job
      variables:
        branchEnvironment: $[replace(variables['Build.SourceBranch'], 'refs/heads/', '')]
        updateConfigurationBoolean: $[stageDependencies.schedule_check.check_if_meets_criteria.outputs['check_if_need_to_schedule.check_updateConfiguration']]
        updateCertificateBoolean: $[stageDependencies.schedule_check.check_if_meets_criteria.outputs['check_if_need_to_schedule.check_updateCertificate']]
        updateSoftwareBoolean: $[stageDependencies.schedule_check.check_if_meets_criteria.outputs['check_if_need_to_schedule.check_updateSoftware']]

      steps:
      - powershell: |
            $branchEnvironment_ps = "$(branchEnvironment)"
            $updateConfigurationBoolean_ps = "$(updateConfigurationBoolean)"
            $updateCertificateBoolean_ps = "$(updateCertificateBoolean)"
            $updateSoftwareBoolean_ps = "$(updateSoftwareBoolean)"

            #Get Date
            $ReportDate = Get-Date -format "MM-dd-yyyy"
            
            #Configuration Variables for E-mail
            $From = {{{ email address to send from }}}
            $To = {{{ email address to send to }}}
            $SMTPServer = {{{ Your SMTP server }}}
            $Subject = "The pipeline {{{ Your pipeline name }}} failed to run on " +$ReportDate
            
            #HTML Template
            $EmailBody = @"
                <html>
                <body style="font-family:calibri;"> 
                    <p>Hello Team,</p>
                    <p>The pipeline failed in the next run(s):</p>
                    <p>Branch : <strong>$branchEnvironment_ps</strong></p>
                    <table style="height: 124px; width: 35.4705%; border-collapse: collapse; border-style: double; border-color: #ff0000; float: left;" border="3">
                    <tbody>
                    <tr style="height: 31px;">
                    <td style="width: 50%; height: 31px;"><strong>Run</strong></td>
                    <td style="width: 50%; height: 31px;"><strong>Failed</strong></td>
                    </tr>
                    <tr style="height: 31px;">
                    <td style="width: 50%; height: 31px;">Configuration </td>
                    <td style="width: 50%; height: 31px;">$updateConfigurationBoolean_ps </td>
                    </tr>
                    <tr style="height: 31px;">
                    <td style="width: 50%; height: 31px;">Certificate</td>
                    <td style="width: 50%; height: 31px;">$updateCertificateBoolean_ps</td>
                    </tr>
                    <tr style="height: 31px;">
                    <td style="width: 50%; height: 31px;">Software</td>
                    <td style="width: 50%; height: 31px;">$updateSoftwareBoolean_ps</td>
                    </tr>
                    </tbody>
                    </table>
                    <p>&nbsp;</p>
                    <div>
                    <div>Please fix this, so the next run can be successful.</div>
                    <div>&nbsp;</div>
                    <div>
                    <div>
                    <div>With kind regards,</div>
                    <div>&nbsp;</div>
                    <div>
                    <div>The {{{Your pipeline name}}} pipeline</div>
                    </div>
                    </div>
                    </div>
                    </div>
                </body>
                </html>
            "@

            Send-MailMessage -From $From -to $To -Subject $Subject -BodyAsHtml $EmailBody -SmtpServer $SMTPServer -UseSsl
        name: send_and_email

```
{% endraw %}

