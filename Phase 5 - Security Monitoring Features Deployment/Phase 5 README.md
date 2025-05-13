# Phase 5: Security Monitoring Features Deployment
In This phase, I will use Terraform to build out a Linux server and install/configure Splunk Enterprise for log aggregation across the AWS environment. I will also create two additional Windows EC2 instances within the private subnet that will act as "workstations" that will be targeted later during our simulated attacks.

## 🎯 Main Target Goals for this Phase:
- Add my Windows EC2 workstations to Active Directory with group policies
- Create a new AWS user with an attatched IAM role that allows them to setup AWS security services
- Create a Terraform script that builds out the following AWS services for monitoring: Guard Duty, Cloud Trail, Cloudwatch, VPC flow logs, AWS config, IAM Acces Analyser

---

# 🔧 Technology Utilized
- Terraform
- AWS EC2 with Windows Server 2022
- Windows AD Domain Controller
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

## ⭐ Step 1️: Adding Windows EC2 Workstations to Active Directory with Groupo Policies

### Creating Sec Group Outbound Rules for Workstations to Communicate with AD Server

I begin by updating the Windows Workstation EC2s with the following inbound/outbound rules within their security groups to ensure proper communication between them: 

#### Workstations OUTBOUND traffic rules to 10.0.2.10 (Windows AD Private IP):
- Port 445 open to 10.0.2.10 (SMB communication)
- Port 53 open to 10.0.2.10 (DNS lookup)
- Port 389 open to 10.0.2.10 (LDAP)
- Port 88 open to 10.0.2.10 (Kerberos)
- Ports 49152-65535 open to 10.0.2.10 (RPC dynamic range)

![image](https://github.com/user-attachments/assets/fee871ee-733e-47ec-8c26-31aeb961a339)

#### Windows AD OUTBOUND traffic rules to 10.0.2.80 AND 10.0.2.90 (Windows Workstations Private IP):
- Port 445 open to 10.0.2.80/10.0.2.90 (SMB communication)
- Port 53 open to 10.0.2.80/10.0.2.90 (DNS lookup)
- Port 389 open to 10.0.2.80/10.0.2.90 (LDAP)
- Port 88 open to 10.0.2.80/10.0.2.90v (Kerberos)
- Ports 49152-65535 open to 10.0.2.80/10.0.2.90 (RPC dynamic range)

![image](https://github.com/user-attachments/assets/11a5a739-da96-479c-b690-212ba78fc33b)


### Setting the Workstation to Join My AD Domain Controller

Once I have the right ports open for communication, my next step is to bastion into my first workstation and change it's domain settings so it can point to  the [Active Directory domain that we created in Phase 3:](https://github.com/ChrisHerrera90/Complete-AWS-Cloud-Security-Architecture-Design-and-Splunk-Detection-Against-Simlulated-Attacks/blob/main/Phase%203:%20Windows%20Active%20Directory%20Server%20EC2%20Instance%20Deployment%20and%20Setup/Phase%203%20README.md#rdp-into-windows-ad-ec2-and-installing-active-directory-and-upgrading-it-to-the-domain-controller) `aws-securityproject.local`

This step will allow our Domain Controller to add this workstation to Group Policies so that I can collect activity logs that will be forwarded to our Splunk server later. In order to do this, I begin by RDPing into my first workstation via my Bastion server and ran the following Powershell command to point the machine to my AD domain's private IP:

`Set-DnsClientServerAddress -InterfaceAlias "Ethernet 3" -ServerAddresses "10.0.2.10"`

This script is meant to point my first workstation EC2s network alias `Ethernet 3` (default for AWS instances) to the AD domain's private IP address `10.0.2.10`

SCREENSHOT

Once I have successfully pointed the domain, I then have to go into my workstation's System Properties --> Change Settings and change the domain to `aws-securityproject.local`

SCREENSHOT

Once I have done this, I then have to restart the workstation. After reboot, I can verify that the domain was changed with the Powershell command `whoami`. In the screenshot below, you can see that the workstation's domain name was successfully changed to

SCREENSHOT

I repeated this process with the second workstation.

---
### Bash Script for Auto-installing Splunk into the Ubuntu Machine
Next, I want to include a BASH script that will auto-download and install Splunk into this EC2 instance. This script is designed to download and install the Splunk version 9.2.1 tarball from Splunk's website, unzips it (`tar -xvzf`) and saves it into the `/opt` folder as a `splunk.tgz` file. Additionally, the script creates a new dedicated `Splunk` user in Ubuntu so that the root user does not have to be used for added security (the `chown` command also gives this user ownership over the Splunk folder). Finally, Splunk is configured to automatically launch at boot.

```tf
  user_data = <<-EOF 
#!/bin/bash
cd /opt
wget -O splunk.tgz https://download.splunk.com/products/splunk/releases/9.2.1/linux/splunk-9.2.1-77f73c9edb85-Linux-x86_64.tgz
tar -xvzf splunk.tgz
useradd splunk
chown -R splunk:splunk splunk
sudo -u splunk /opt/splunk/bin/splunk start --accept-license --answer-yes --no-prompt
sudo -u splunk /opt/splunk/bin/splunk enable boot-start
EOF

```


---
### Creating a Variable Key for SSH Access to Ubuntu Splunk EC2

Next, I need to create a variable key that can be used to log into the Windows EC2 server via RDP securely. I also created the `terraform.tfvars` file which holds the key values that will populate in the instance (`terraform.tfvars` file is redacted for security reasons)

```tf
variable "key_name" {
  description = "Name of the AWS key pair to access the Ubuntu Splunk EC2 Instance"
  type = string
}
  
```

---
### 👀 Executing and Verifying The Terraform Scripts Where Successful (AWS Dashboard)
After writing these scripts, I went ahead and ran them in my Visual Studio Terminal and checked if Terraform validated my code with `terraform validate`. With a successful validation, I went ahead and planned and executed the build by using the `terraform apply` command. Once initiated, we receive a message saying that our resources were successfully created:

![image](https://github.com/user-attachments/assets/fec737b8-3266-4617-a936-9ebb3e3d9735)


If successful, the Terraform script execution should have resulted in the following AWS creations:
- Security Group (ubuntusplunk_secgroup)
- Ubuntu EC2 Instance (Ubuntu-Splunk-EC2)
- Private IP assigned to the Ubuntu EC2
- Public IP assigned to Ubuntu EC2

Once the commands executed, I went to my AWS dashboard to confirm that all of the requested resources and configurations were implemented. 

![image](https://github.com/user-attachments/assets/c5f76a62-b7fa-4429-ad5b-d02c322e8a25)




---
### 👀 Logging into the Ubuntu Server with SSH and Verifying Splunk Installation + Splunk User Creation

Once the Ubuntu EC2 instance was created, I went ahead and SSH'd into it via my Bastion server (using the access key `.pem.` file) to verify that Splunk was installed and that the Splunk user was created as the owner of the Splunk file. However, Splunk did not download and install correctly due to a networking issue (had to open up NACLs) and because I used the incorrect wget url link. So I had to manually download and install it::


![image](https://github.com/user-attachments/assets/6d6e7e32-f93c-4c59-a5b6-3ecccda358bc)
![image](https://github.com/user-attachments/assets/dcd7f9d6-acae-43c7-b598-5b5adf819f74)

Once I extracted and installed Splunk, I went ahead and set it to boot at startup so it runs whenever the Ubuntu machine is running:

![image](https://github.com/user-attachments/assets/e7d2e267-ed51-403a-8197-2a2340815f03)


---
## ⭐ Step 2: Breakdown of `Windows Workstation EC2 Deployment.tf` Terraform Script

### Creating Two Windows Workstations that Will Be Targetted for Attacks Later
Now that we have our Ubuntu Splunk server installed, I am now going to build two Windows workstations within my private subnet that will be targeted during my attack simulations in the later phases of this project. The Terraform scripts will be very similar to our Windows AD EC2 server build, however, keep in mind that security group settings will most likely be altered in later phases for attack simulation purposes.

The Terraform script for these two Windows Workstations EC2s are almost identical to the [Phase 3 Windows AD Server EC2](https://github.com/ChrisHerrera90/Complete-AWS-Cloud-Security-Architecture-Implementation-and-Testing-against-a-Simulated-Attack/blob/main/Phase%203%3A%20Windows%20Active%20Directory%20Server%20EC2%20Instance%20Deployment%20and%20Setup/Phase%203%20README.md#creating-the-ec2-instance-that-will-become-the-windows-ad-domain-controller). The only difference is the removal of the "Kerberos" ingress rule.

After validating and applying the Terraform commands, I went into the AWS dashboard to confirm the creation of both EC2s and their respective security groups:

![image](https://github.com/user-attachments/assets/fe11b267-4abf-494f-aefd-1439bdcfc96e)
![image](https://github.com/user-attachments/assets/c476cb04-47c6-4988-8df3-652ae71cffc6)



## Now that I have created the Splunk and Workstations servers, we are ready for [Phase 5 - Security Monitoring Features Deployment](https://github.com/ChrisHerrera90/Complete-AWS-Cloud-Security-Architecture-Design-and-Splunk-Detection-Against-Simlulated-Attacks/blob/main/Phase%205%20-%20Security%20Monitoring%20Features%20Deployment/Phase%205%20README.md)
