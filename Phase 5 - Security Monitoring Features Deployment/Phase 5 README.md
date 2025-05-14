# Phase 5: Security Monitoring Features Deployment
In This phase, I will use Terraform to build out a Linux server and install/configure Splunk Enterprise for log aggregation across the AWS environment. I will also create two additional Windows EC2 instances within the private subnet that will act as "workstations" that will be targeted later during our simulated attacks.

## 🎯 Main Target Goals for this Phase:
- Add my Windows EC2 workstations to Active Directory with group policies
- Create a new AWS user with an attatched IAM role that allows them to setup AWS security services
- Create a Terraform script that builds out the following AWS services for monitoring: Guard Duty, Cloud Trail, Cloudwatch, VPC flow logs, AWS config, IAM Acces Analyser

---

## 🔧 Technology Utilized
- Terraform
- AWS EC2 with Windows Server 2022
- Windows AD Domain Controller
- Active Directory and Group Policies
- AWS Security Groups
- Sysmon
- AWS Guard Duty, Cloud Trail, Cloudwatch, VPC flow logs, AWS config, and IAM Acces Analyser

---

## 📽️ Video: Lessons Learned

[![Watch the video](https://img.youtube.com/vi/TyxBAQfMKmw/0.jpg)](https://youtu.be/TyxBAQfMKmw)

---

# Table of Contents

- [Creating the VPC Security Group for My Ubnuntu Splunk Server](#creating-the-vpc-security-group-for-my-ubnuntu-splunk-server)
- [Creating the EC2 Instance That Will Become the Ubuntu Splunk Server](#creating-the-ec2-instance-that-will-become-the-ubuntu-splunk-server)
- [Bash Script for Auto-installing Splunk into the Ubuntu Machine](#bash-script-for-auto-installing-splunk-into-the-ubuntu-machine)
- [Creating a Variable Key for SSH Access to Ubuntu Splunk EC2](#creating-a-variable-key-for-ssh-access-to-ubuntu-splunk-ec2)
- [Executing and Verifying The Terraform Scripts Where Successful (AWS Dashboard)](#-executing-and-verifying-the-terraform-scripts-where-successful-aws-dashboard)
- [Logging into the Ubuntu Server with SSH and Verifying Splunk Installation + Splunk User Creation](#-logging-into-the-ubuntu-server-with-ssh-and-verifying-splunk-installation--splunk-user-creation)
- [Creating Two Windows Workstations that Will Be Targetted for Attacks Later](#creating-two-windows-workstations-that-will-be-targetted-for-attacks-later)

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












---
---   
## ⭐ Step 3: Configuring AWS Security Monitoring Services with Terraform


## Now that I have created the Splunk and Workstations servers, we are ready for [Phase 5 - Security Monitoring Features Deployment](https://github.com/ChrisHerrera90/Complete-AWS-Cloud-Security-Architecture-Design-and-Splunk-Detection-Against-Simlulated-Attacks/blob/main/Phase%205%20-%20Security%20Monitoring%20Features%20Deployment/Phase%205%20README.md)
