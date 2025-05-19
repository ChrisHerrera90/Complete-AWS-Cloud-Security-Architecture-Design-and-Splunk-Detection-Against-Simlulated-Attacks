# Phase 5: Security Monitoring Features Deployment
In This phase, I will use Terraform to build out a Linux server and install/configure Splunk Enterprise for log aggregation across the AWS environment. I will also create two additional Windows EC2 instances within the private subnet that will act as "workstations" that will be targeted later during our simulated attacks.

## 🎯 Main Target Goals for this Phase:
- Add my Windows EC2 workstations to Active Directory with group policies
- Create a new AWS user with an attached IAM policy that allows them to set up AWS security services
- Create a IAM role for my Splunk EC2 instance to allow it to pull logs from these AWS services
- Configure the following AWS services for monitoring: Guard Duty, Cloud Trail, Cloudwatch, VPC flow logs, IAM Acces Analyser

---

## 🔧 Technology Utilized
- Terraform
- AWS EC2 with Windows Server 2022
- Windows AD Domain Controller
- Active Directory and Group Policies
- AWS Security Groups
- Sysmon
- AWS Guard Duty, Cloud Trail, Cloudwatch, VPC flow logs, and IAM Acces Analyser

---

## 📽️ Video: Lessons Learned

[![Watch the video](https://img.youtube.com/vi/TyxBAQfMKmw/0.jpg)](https://youtu.be/TyxBAQfMKmw)

---

# Table of Contents

- [Creating Sec Group Outbound rule for workstations to communicate with AD Server](#creating-sec-group-outbound-rules-for-workstations-to-communicate-with-ad-server)
- [Adding Workstations to my Domain Controller](#setting-the-workstation-to-join-my-ad-domain-controller)
- [Installing Sysmon on all my EC2s for Logging](#installing-sysmon-on-my-4-windows-ec2s)
- [Creating AD Group Policy Objects for Windows Workstations](#creating-ad-group-policies-for-my-windows-workstations)
- [Creating a new IAM User with permissions to manage AWS monitoring services](#-step-2-creating-a-new-iam-user-with-permissions-to-configure-aws-security-monitoring-services)
- [Configuring AWS monitoring services withing AWS dashboard](#-step-3-configuring-aws-security-monitoring-services)


---

## ⭐ Step 1️: Adding Windows EC2 Workstations to Active Directory with Group Policies

### Creating Sec Group Outbound Rules for Workstations to Communicate with AD Server

I begin by updating the Windows Workstation EC2s with the following inbound/outbound rules within their security groups to ensure proper communication between them: 

#### Workstations OUTBOUND traffic rules to 10.0.2.10 (Windows AD Private IP):
- Port 445 open to 10.0.2.10 (SMB communication)
- Port 53 open to 10.0.2.10 (DNS lookup)
- Port 389 open to 10.0.2.10 (LDAP)
- Port 88 open to 10.0.2.10 (Kerberos)
- Ports 49152-65535 open to 10.0.2.10 (RPC dynamic range)
- All traffic open to 10.0.2.10 

![image](https://github.com/user-attachments/assets/cb80c7b0-4668-4e81-8166-e2ae3fde64b9)


#### Windows AD OUTBOUND traffic rules to 10.0.2.80 AND 10.0.2.90 (Windows Workstations Private IP):
- Port 445 open to 10.0.2.80/10.0.2.90 (SMB communication)
- Port 53 open to 10.0.2.80/10.0.2.90 (DNS lookup)
- Port 389 open to 10.0.2.80/10.0.2.90 (LDAP)
- Port 88 open to 10.0.2.80/10.0.2.90v (Kerberos)
- Ports 49152-65535 open to 10.0.2.80/10.0.2.90 (RPC dynamic range)

![image](https://github.com/user-attachments/assets/11a5a739-da96-479c-b690-212ba78fc33b)
![image](https://github.com/user-attachments/assets/66044946-d20f-461e-a370-1ac244e84b02)

---

### Setting the Workstation to Join My AD Domain Controller

Once I have the right ports open for communication, my next step is to bastion into my first workstation and change it's domain settings so it can point to the [Active Directory domain that we created in Phase 3:](https://github.com/ChrisHerrera90/Complete-AWS-Cloud-Security-Architecture-Design-and-Splunk-Detection-Against-Simlulated-Attacks/blob/main/Phase%203:%20Windows%20Active%20Directory%20Server%20EC2%20Instance%20Deployment%20and%20Setup/Phase%203%20README.md#rdp-into-windows-ad-ec2-and-installing-active-directory-and-upgrading-it-to-the-domain-controller) `aws-securityproject.local`

This step will allow our Domain Controller to add this workstation to Group Policies so that I can collect activity logs that will be forwarded to our Splunk server later. In order to do this, I begin by RDPing into my first workstation via my Bastion server and ran the following Powershell command to point the machine to my AD domain's private IP:

`Set-DnsClientServerAddress -InterfaceAlias "Ethernet 3" -ServerAddresses "10.0.2.10"`

This script is meant to point my first workstation EC2s network alias `Ethernet 3` (default for AWS instances) to the AD domain's private IP address `10.0.2.10`

![image](https://github.com/user-attachments/assets/eb297297-666e-448b-9422-8ce6708aaaf1)


Once I have successfully pointed the domain, I then have to go into my workstation's `System Properties` --> `Advanced System Settings` and change the domain to `aws-securityproject.local`

![image](https://github.com/user-attachments/assets/d4ee7bca-663b-46f4-8185-2e698ad0058d)

![image](https://github.com/user-attachments/assets/a615106b-7466-49bb-bc0d-fb9ef9d30aa0)

![image](https://github.com/user-attachments/assets/377c2035-669b-4f72-bda6-aa5426abfde3)

![image](https://github.com/user-attachments/assets/738d3280-b902-4ceb-94f8-3cd3b8c50867)

![image](https://github.com/user-attachments/assets/8001ff1d-f96f-4e6b-9627-efcc016699c9)


Once I have done this, I then have to restart the workstation. After reboot, I can verify that the domain was changed in the system properties tab. In the screenshot below, you can see that the workstation's domain name was successfully changed to `EC2AMAZ-FB04MOV.aws-securityproject.local`

![image](https://github.com/user-attachments/assets/3440c072-f89f-4d8b-aa35-fed73db6926b)

I repeated this process with the second workstation.

---

### Installing Sysmon on my 4 Windows EC2s

Before I configure my Group Policies in AD, I will need to install Sysmon (System Monitor) on all of my Windows EC2s so that I can log detailed events about process creation, network connections, and file changes across my environment. This will help supplement data that standard Windows Event Logs do not provide.

The process of installing Sysmon will be the same across all 3 of my Windows machines. The steps involved the following:
1. Download Sysmon onto my machine
2. Download the `SwiftOnSecurity Sysmon Config` file (I will use this for the security configuration of sysmon logging)
3. Run the following Powershell command to install it in the `C:\Tools\Sysmon` directory: `.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml`
4. Verify that Sysmon is running by using the following Powershell command: `Get-Service sysmon64`
5. Verify Sysmon is collecting event logs by going to `Event Viewer > Applications and Services Logs > Microsoft > Windows > Sysmon > Operational`


See the screenshots below for an example install on one of my Windows Workstations:

![image](https://github.com/user-attachments/assets/ae939f2a-af91-4c11-b277-b331f7f35751)

![image](https://github.com/user-attachments/assets/90c7fed8-57ce-4267-b681-560456b83a1f)

![image](https://github.com/user-attachments/assets/1ee009b1-d35e-4d8e-951a-298232993dff)

![image](https://github.com/user-attachments/assets/292345db-a294-4a50-82d9-e9870f90847c)

---

### Creating AD Group Policies for my Windows Workstations

In order to start retrieving event logs from my Workstations for my simulated attacks, I begin by creating a new Organizationl Unit (OU) within my `aws-securityproject.local` domain. I will call it `EC2 Workstations OU`:

![image](https://github.com/user-attachments/assets/4c9ef669-3120-4a47-824f-e3927f828a2f)

Then, I pulled up `Active Directory Users and Computers`, searched for my 2 Windows workstations, and "moved" them to this new organizational unit:

![image](https://github.com/user-attachments/assets/e64d5b5d-4b36-4e15-a5f9-5aedef86d8bd)

![image](https://github.com/user-attachments/assets/9d85e075-c57b-4154-b53b-f87e50633517)


Once I have added my workstations, I am going to next create a new `Group Policy Object` (GPO) within this new OU and name it `Security Logging for Splunk`:

![image](https://github.com/user-attachments/assets/f8833208-fdb0-4ab7-9940-00e9a8881ba4)

### Once the new GPO is created, I will then edit it to include the following policies:

#### ✅ Enable Advanced Auditing 
Group Policy Management Editor Location: `Computer Configuration > Policies > Windows Settings > Security Settings > Advanced Audit Policy Configuration > Audit Policies`
Subcategories enabled:
Logon/Logoff
- Logon
- Logoff
- Special Logon
- Other Logon/Logoff Events

Account Logon
- Credential Validation
- Kerberos Authentication Service
- Kerberos Service Ticket Operations

Detailed Tracking
- Process Creation
- PNP Activity

Object Access
- File System
- Registry

Policy Change
- Audit Policy Change

Account Management
- User Account Management
- Group Membership


![image](https://github.com/user-attachments/assets/12559963-2ecf-4f9f-bbd6-8a5e3014946d)


#### ✅ Enable Command Line Logging
Group Policy Management Editor Location: `Computer Configuration > Administrative Templates > System > Audit Process Creation`

Here, we travel to the `Audit Process Creation` sub-folder within Group Policy Management Editor and enable `include command line creation process events` so that Splunk can receive command line events via `Event ID 4688`

![image](https://github.com/user-attachments/assets/c9c3fb62-def9-4ea3-8efd-5f3b521a8e73)


#### ✅ Enable Powershell Logging
Group Policy Management Editor Location: `Computer Configuration > Administrative Templates > Windows Components > Windows PowerShell`

Here, we travel to the `Windows Powershell` sub-folder within Group Policy Management Editor and enable `Turn on PowerShell Script Block Logging` and `Turn on PowerShell Transcription` so that Splunk can receive Powershell events via `Event ID 4103, 4104, 600`

![image](https://github.com/user-attachments/assets/aadb809e-577a-43d7-9a2c-a995c9712293)

#### ✅ Enable Microsoft Defender Antivirus Logging
Group Policy Management Editor Location: `Computer Configuration > Administrative Templates > Windows Components > Microsoft Defender Antivirus > Real-Time Protection`

Here, we travel to the `Reporting` sub-folder of Microsoft Defender within Group Policy Management Editor and enable `Monitor file and program activity on your computer` and `Scan all downloaded files and attachments` so that Splunk can receive thesde Defender event logs:

![image](https://github.com/user-attachments/assets/4ff33feb-74fe-411c-96c0-4819c970b383)


---
---
## ⭐ Step 2: Creating a new IAM User with Permissions to Configure AWS Security Monitoring Services

Now that I have my Active Directory set up for my workstations, my next step is to create a new IAM policy, IAM role for my Splunk EC2 and a user that will be in charge of setting up the AWS monitoring services (Cloudwatch, Guardduty, etc.). I will accomplish this by creating a new Terraform script (`Splunk User IAM Monitoring Setup.tf`)  for deployment in AWS. Here is the breakdown of the Terraform script:

### ☑️ Creating the IAM Policy in Terraform

I first began by creating the `IAM Policy` JSON file that will allow my user to have access to the following AWS services needed to monitor my environment and generate logs for my Splunk server:
- Guard Duty
- Cloud Trail
- Cloud Watch
- VPC Flow logs
- IAM Access Analyzer

```tf
resource "aws_iam_policy" "splunk_log_monitoring_policy" {
  name = "SplunkLogMonitoringAccessPolicy"
  description = "IAM policy for Splunk log monitoring access"
  path = "/"
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Action = [
          "guardduty:*",
          "cloudtrail:*",
          "logs:*",
          "ec2:DescribeFlowLogs",
          "ec2:CreateFlowLogs",
          "config:*",
          "access-analyzer:*"
        ],
        Resource = "*"
      }
    ]
  })
}

```

Once the Terraform was deployed, I was able to verify that the IAM policy was created:

![image](https://github.com/user-attachments/assets/a63d8199-f17c-41eb-bed4-a047208a026b)


### ☑️ Creating the IAM Role for my Splunk EC2 in Terraform

Once I set up the IAM policy that my user needs, I then have to create an `IAM Role`, attach it to the IAM policy, and create an instance profile that I can 
then attach to my Splunk EC2 (using BASH) to allow it the ability to pull logs from the above AWS services. I will then manually connect this IAM role to my Splunk EC2 in the dashboard: 

```tf
resource "aws_iam_role" "splunk_log_monitoring_ec2_role" {
  name = "SplunkLogMonitoringEC2Role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect = "Allow",
        Principal = {
          Service = "ec2.amazonaws.com"
        },
        Action = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "attach_policy_to_role" {
  role = aws_iam_role.splunk_log_monitoring_ec2_role.name
  policy_arn = aws_iam_policy.splunk_log_monitoring_policy.arn
}

resource "aws_iam_instance_profile" "splunk_instance_profile" {
  name = "splunk-log-monitoring-instance-profile"
  role = aws_iam_role.splunk_log_monitoring_ec2_role.name
}

```

Once the Terraform was deployed, I was able to verify that the IAM role was created:

![image](https://github.com/user-attachments/assets/d335615f-af38-44ae-95ef-c52a27be3106)



### **🚨 Future Note:** 

After I deployed this terraform script, I then had to run this BASH command in the Splunk EC2 CLI in order to manually attach the newly created resource instance profile/role/policy to it. This is the last step that will allow my Splunk server to pull any generated logs from AWS:

```
aws ec2 associate-iam-instance-profile \
  --instance-id i-03441db2cff0e7730 \
  --iam-instance-profile Name="splunk-log-monitoring-instance-profile"

```

![image](https://github.com/user-attachments/assets/8057f2c1-f125-4dc9-8f88-5fc7fcb1fd01)





### ☑️ Creating the User that Will Configure the AWS Services

Finally, we are going to create the User `Splunk AWS Monitoring` that will have our `splunk_log_monitoring_policy` attached to it so we can utilize it to configure the AWS monitoring services

```
resource "aws_iam_user" "splunk_aws_monitoring_user" {
  name = "splunk_aws_Monitoring"
}

resource "aws_iam_user_policy_attachment" "attach_policy_to_user" {
  user = aws_iam_user.splunk_aws_monitoring_user.name
  policy_arn = aws_iam_policy.splunk_log_monitoring_policy.arn
}

```
Once the Terraform was deployed, I was able to verify that the IAM User was created:

![image](https://github.com/user-attachments/assets/f36f0f3d-0a30-4228-9c57-d84fbcc149c5)


---
---   
## ⭐ Step 3: Configuring AWS Security Monitoring Services

Finally, we are going to end this phase with configuring the following AWS security monitoring services that will generate logs for Splunk:
- Guard Duty
- CloudTrail
- Cloud Watch
- VPC flow logs
- IAM Access Analyzer

Let's begin!

### ☑️ Configuring Guard Duty

We will begin with Configuring GuardDuty which is a threat detection service that uses AI and machine learning to detect threats such as port scanning, AWS account compromises, malware and suspicious networking activities within my EC2 instances, ransomware and more. The cool thing about GuardDuty is that it builds its threat detection capabilities automatically based on threat intelligence feeds!

To enable GuardDuty in my AWS environment, I looked it up in the AWS dashboard and enabled all the features. No other configuration was necessary since it automatically starts to monitor my environment. Easy!

![image](https://github.com/user-attachments/assets/2ed2511c-2924-4305-ad29-ca98241cee8d)
![image](https://github.com/user-attachments/assets/1f77e5cc-f3bb-493a-b810-ec3f907311ea)
![image](https://github.com/user-attachments/assets/cd3dddc7-bc61-4751-96c1-c083fc302bb3)


### ☑️ Configuring CloudTrail

CloudTrail is an AWS service that basically monitors all activity within my AWS cloud environment. It will record all actions taken by users, roles and AWS services that are active in my environment.

To begin, I first had to navigate to the CloudTrail service and click `create trail`. I will be naming this trail `SplunkTrail`, additionally, a S3 bucket will automatically be created to save these logs:

![image](https://github.com/user-attachments/assets/a0636999-3bac-49db-bcb0-ca0a94d8a4dd)
![image](https://github.com/user-attachments/assets/5adad7a8-3ec4-4300-90fd-377e925c3460)
![image](https://github.com/user-attachments/assets/29c9367f-fa94-492f-be82-d48c8c8d21fa)

Additionally, I am going to configure my `SplunkTrail` to enable a `CloudWatch log integration` so that any logs generated can be sent to CloudWatch, which in turn, will forwarded to Splunk later on:

![image](https://github.com/user-attachments/assets/c539a00e-590b-4058-9287-585d6a37979d)
![image](https://github.com/user-attachments/assets/1a0787e6-d31b-4a71-84b3-2806b3655a49)


### ☑️ Configuring Cloud Watch

Next, I want to configure CloudWatch. CloudWatch is basically a centralized place within AWS where you can monitor logs, events and performance metrics across my entire AWS environment. CloudWatch also allows the ability to create pre-defined alarms to alert me to any suspicious activity, as well as automated actions that can be taken to respond to these alerts. For now, I will use this service to act as the main aggregator of logs generated from across the environment, which will then be forwarded to Splunk: 

To begin, we will navigate to `CloudWatch` and begin creating `log groups` from all the sources of logs within my environment with a retention setting of 30 days. Sources of logs will include:
- CloudTrail logs (done in previous step)
- VPC flow logs
- Sysmon logs from all EC2s
- Active Directory event logs
- Note: IAM Access Analyzer is not compatible with CloudWatch at this time

Here is an example of the `VPC Flowlogs` Cloudwatch Log Group:
![image](https://github.com/user-attachments/assets/ce8be4b7-d6c6-4228-b818-2d9ecd85b088)


### ☑️ Configuring VPC flow logs

Next, I configured the VPC flow logs service. This service actively monitors all network traffic across my AWS networking environment (VPC, subnets, route tables, gateways, etc.). This will be crucial in identifying any suspicious network activity such as data exfiltration, port scans, lateral movement within my private network, and C2 traffic).

Creating the VPC flow logs was fairly straightforward. I went to my VPC service, clicked on `Create Flow Log` and configure it to do the following:
- Filter `ALL` traffic
- Destination of logs = `CloudWatch Logs`
- Log group = `splunk-aws-VPC-flowlogs`
- Maximum aggregation interval= 1 minute
- IAM role = `SplunkLogMonitoringEC2Role` (we created this earlier in Step 2)

I created flow logs for both my public and private subnet:

![image](https://github.com/user-attachments/assets/558329e1-c2c9-4123-be22-f4bdca4a0048)
![image](https://github.com/user-attachments/assets/8602e4b5-2416-430f-a40f-34ab39bda23f)

### ☑️ Configuring IAM Access Analyzer

IAM Access Analyzer tracks IAM policies across all of my AWS resources to monitor for unintended or malicious changes to IAM policies that would allow a malicious attacker unauthorized access to my resources.

Setup was very simple, I just needed to navigate to the IAM Access Analyzer resource and create a new "analyzer" that is set to monitor/scope my entire AWS account:

![image](https://github.com/user-attachments/assets/e5d00329-6123-4bfc-81b3-f1accbb20ecb)
![image](https://github.com/user-attachments/assets/c3d0b280-0dd4-48c2-b14d-83ea06e10e6a)


---
--



## Now that I have set up my monitoring and detection services/tools, we are ready for [Phase 6 - Splunk Log Ingestion Setup](https://github.com/ChrisHerrera90/Complete-AWS-Cloud-Security-Architecture-Design-and-Splunk-Detection-Against-Simlulated-Attacks/blob/main/Phase%206%20-%20Splunk%20Log%20Ingestion%20Setup/Phase%206%20README.md)
