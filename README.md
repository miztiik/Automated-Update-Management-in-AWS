# AWS Systems Manager Patch Manager

Automation through Patch Manager & Maintenance Window services allows us to manage operating system updates and upgrades for Windows and Linux computers implemented in AWS, on-premises environments or other cloud providers. We can quickly review the status of available updates on all agent computers and manage the process for installing required updates for servers.

### Pre-Requisites
- Create a Default Patch Baseline<sup>1</sup>
- Add Instances to a Patch Group
- Create a Maintenance Window for Patching
- IAM Managed Roles:
  1. `AmazonEC2RoleforSSM` - All EC2 instances need to this permission to be attached.
  1. `AmazonSSMMaintenanceWindowRole` - Not down the ARN, We will need this ARN to configure maintenance window.
- Make sure all EC2 instances to be patched have AWS SSM Agent Installed and communicating with SSM
  1. Requires _outbound_ `443` to internet SG Rule


## Configure Patch Baseline
AWS provides default baselines for all major OS's as recommend by the ISVs, If needed we can customize these patch baselines for inclusion or exclusion of any specific patches. The default configuration is good place to start with in most use cases. The same can be duplicated and used for your clients, and made the default baseline to allow for future customizations.

![Configure Patch Baseline](https://raw.githubusercontent.com/miztiik/Automated-Update-Managed-in-AWS/master/images/ssm-patch-management-00.png)

### Add instances to a patch group
To allow fine control over the patching of virtual machines running different OS's(example Windows 2016, Windows 2012, RHEL7.x, Amazon Linux etc) and different business functions like (Webservers running IIS or Apache or nginx etc) each of them need to have a special tag key `Patch Group` _DO NOT CUSTOMIZE TAG; USE AS-IS_

| Tag Key     | Tag Value          | Effect                                                                                               | Schedule                  |
|-------------|--------------------|------------------------------------------------------------------------------------------------------|---------------------------|
| Patch Group | Windows            | **ALL** windows flavors will fall into this category                                                 |                           |
| Patch Group | RHEL               | **ALL** Redhat flavors will fall into this category                                                  |                           |
| Patch Group | Windows2012        | **ALL** Windows 2012 servers will fall into this category **ALL** Security patches will be installed |                           |
| Patch Group | Management Server  | **ALL** Shared Service Servers                                                                       | cron(00 23-5 ? * TUE#1 *) |
| Patch Group | Web Server         | **ALL** Web Servers                                                                                  | cron(00 23-5 ? * THU#1 *) |
| Patch Group | Application Server | **ALL** Application Servers                                                                          | cron(00 23-5 ? * THU#2 *) |
| Patch Group | Database Server    | **ALL** Database Servers                                                                             | cron(00 23-5 ? * THU#3 *) |

*The advantage of Thursday patching is that people are available on Friday in case something should not have gone well. _Customize as required_.

### Maintenance Windows
To minimize the impact on service availability, It is recommend that we configure a Maintenance Window to execute patching during times that won't interrupt business operations. AWS Systems Manager Maintenance Windows let you define a schedule for when to perform potentially disruptive actions on your instances such as patching an operating system, updating drivers, or installing software or patches. Each Maintenance Window has a schedule, a duration, a set of registered targets.
< 2 images>
![Configure Maintenance Windows](https://raw.githubusercontent.com/miztiik/Automated-Update-Managed-in-AWS/master/images/ssm-patch-management-01.png)

#### Maintenance Window Run Task
Choose the maintenance windows that suits your environment needs. You can choose a [cron expression](https://docs.aws.amazon.com/systems-manager/latest/userguide/reference-cron-and-rate-expressions.html?shortFooter=true). _Customize as required_.

| Environment | Patching On  | Time       |
|-------------|--------------|------------|
| Sandbox     | Weekdays     | 10PM-3AM   |
| Development | Weekdays     | 10PM-3AM   |
| Test        | Weekdays     | 10PM-3AM   |
| Acceptance  | Weekend      | 6AM - 11AM |
| Production  | Weekend      | 6AM - 11AM |

![Configure Maintenance Schedules](https://raw.githubusercontent.com/miztiik/Automated-Update-Managed-in-AWS/master/images/ssm-patch-management-02.png)

_Important:_ After installing patches, Systems Manager reboots each instance. The reboot is required to make sure that patches are installed correctly and to ensure that the system did not leave the instance in a potentially bad state.

### Add Targets to Maintenance window
The easiest way to attach targets is to use the tag key `Patch Group` and have multiple values suiting your needs.

Create `Patch Group` for your environment, For example,
- `Win` - For patching all windows Servers
- `Linux` - For all servers running Linux OSes
- `Web Servers` - For pathing all Web Servers
![Patch Group](https://raw.githubusercontent.com/miztiik/Automated-Update-Managed-in-AWS/master/images/ssm-patch-management-03.png)
![Register Targets](https://raw.githubusercontent.com/miztiik/Automated-Update-Managed-in-AWS/master/images/ssm-patch-management-04.png)

### Add `Runtask` to patch the servers
![Add Tasks](https://raw.githubusercontent.com/miztiik/Automated-Update-Managed-in-AWS/master/images/ssm-patch-management-05.png)
`AWS-RunPatchBaseline` - This is the latest version of document [recommended](https://docs.aws.amazon.com/systems-manager/latest/userguide/patch-manager-ssm-documents.html#patch-manager-ssm-documents-recommended-AWS-InstallWindowsUpdates) by Amazon to install/scan patches.
![Add Run Document](https://raw.githubusercontent.com/miztiik/Automated-Update-Managed-in-AWS/master/images/ssm-patch-management-06.png)
![Add Run Document](https://raw.githubusercontent.com/miztiik/Automated-Update-Managed-in-AWS/master/images/ssm-patch-management-07.png)

**You can control how many instances are patched at any given time and how to handle errors using `Rate Control`**
![Rate Control](https://raw.githubusercontent.com/miztiik/Automated-Update-Managed-in-AWS/master/images/ssm-patch-management-08.png)
Update the Role ARN, that was created in the _Pre-Requisites_.
##### Save Output to S3
In case you want to analyze the patch update logs, you can save them to S3
![Save Output to S3](https://raw.githubusercontent.com/miztiik/Automated-Update-Managed-in-AWS/master/images/ssm-patch-management-09.png)

### `Scan` or `Install`
If you want to only `Scan` your instances for missing patches then use the `Scan` option, If you want the patches to be installed, use the `Install` Option. _The below screenshot shows only _scan_.
![Scan for missing patches](https://raw.githubusercontent.com/miztiik/Automated-Update-Managed-in-AWS/master/images/ssm-patch-management-10.png)

**Now that all the steps are completed. The automation will kick in as per the schedule chosen.**

### Monitoring Updates 
Under managed instances of SSM, we can find all the missing and installed patches. Under compliance section the overall view can be seen.
![Save Output to S3](https://raw.githubusercontent.com/miztiik/Automated-Update-Managed-in-AWS/master/images/ssm-patch-management-11.png)



###### Cron References
| Example                   | Details                                                            |
|---------------------------|--------------------------------------------------------------------|
| cron(00 23-5 ? * THU#3 *) | Run third Thursday of every month between 23:00 and 05:00 UTC time |
| cron(0 2 ? * THU#3 *)     | 02:00 AM the third Thursday of every month                         |
| cron(15 10 ? * * *)       | 10:15 AM every day                                                 |
| cron(15 10 ? * MON-FRI *) | 10:15 AM every Monday, Tuesday, Wednesday, Thursday and Friday     |
| cron(0 2 L * ? *)         | 02:00 AM on the last day of every month                            |
| cron(15 10 ? * 6L *)      | 10:15 AM on the last Friday of every month                         |


#### References
1. [Create a Default Patch Baseline](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-patch-baseline-console.html)



1. [Preventing blacklisted applications with AWS Systems Manager and AWS Config](https://aws.amazon.com/blogs/mt/preventing-blacklisted-applications-with-aws-systems-manager-and-aws-config)





