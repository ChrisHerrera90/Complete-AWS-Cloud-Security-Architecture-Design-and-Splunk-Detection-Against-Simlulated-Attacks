 # Phase 4: Deploying the EC2 Splunk Server and Target Workstations
In This phase, I will use Terraform to build out a Linux server and install/configure Splunk Enterprise for log aggregation across the AWS environment. I will also create two additional Windows EC2 instances within the private subnet that will act as "workstations" that will be targeted later during our simulated attacks.

## 🎯 Main Target Goals for this Phase:
- Create a Linux EC2 to install and deploy Splunk in the public subnet.
- Create two additionally Windows EC2 workstations in the private subnet.
- Configure all relevant security groups and NACLs for all instances.
  
---

# 🔧 Technology Utilized
- Terraform
- AWS EC2 with Windows Server 2022
- AWS EC2 with Ubuntu 22.04
- Splunk
- Private and private VPC subnets
- VPC NACL
- RDP (Remote Desktop Protocol)

  
---


# Table of Contents

- [Creating the VPC Security Group for My Windows Domain Controller EC2](#creating-the-vpc-security-group-for-my-windows-domain-controller-ec2)
- [Creating the EC2 Instance That Will Become the Windows AD Domain Controller](#creating-the-ec2-instance-that-will-become-the-windows-ad-domain-controller)
- [Creating a Variable Key for RDP Access to Windows AD EC2](#creating-a-variable-key-for-rdp-access-to-windows-ad-ec2)
- [Creating a Bastion Host (Jump Box) to Log into the Private EC2 via RDP](#creating-a-bastion-host-jump-box-to-log-into-the-private-ec2-via-rdp)
- [Creating The Bastion Host EC2 Security Group so I can RDP Into It](#creating-the-bastion-host-ec2-security-group-so-i-can-rdp-into-it)
- [Creating the EC2 Instance That Will Become the Bastion Host (Jump Host)](#creating-the-ec2-instance-that-will-become-the-bastion-host-jump-host)
- [Setting Up a Public and a Private NACL so that Bastion and Windows AD can Communicate via RDP](#setting-up-a-public-and-a-private-nacl-so-that-bastion-and-windows-ad-can-communicate-via-rdp)
- [RDP Into Windows AD EC2 and Installing Active Directory and Upgrading it to the Domain Controller](#rdp-into-windows-ad-ec2-and-installing-active-directory-and-upgrading-it-to-the-domain-controller)


---

## ⭐ Step 1️: </> Breakdown of `Ubuntu 22.04 Splunk EC2 Creation.tf` for Linux Splunk Server EC2 Creation

### Creating the VPC Security Group for My Ubnuntu Splunk Server

I begin by creating the EC2 instance security group and naming it `ubuntusplunk_secgroup` with the following inbound/outbound rules: 

`ingress` (inbound traffic) rules:
- Allow open ports for SSH exclusively for my Bastion's private IP
- Allow HTTP traffic from the default Splunk web interface port 8000. This will allow me to log into my Splunk account via this Ubuntu VM.
- Allow this Ubuntu machine to receive logs via port 9991 from any resource within the private network.

`egress` (outbound traffic) rules:
- Allow this security group to reach the internet and communicate with HTTP and HTTPS . This will be needed to download the Splunk software, get future updates, etc.

Finally, I made sure to include some `data` blocks to ensure that any values such as the `ami`, `aws_vpc`, and `aws_subnet` was pulling values from our VPC setup in Phase 2.

```tf
provider "aws" {
  region = "us-east-1"
}

data "aws_ami" "ubuntu_server" {
  most_recent = true
  owners      = ["099720109477"] 

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }
}

data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["AWSsecuritylab-VPC"] 
  }
}  

data "aws_subnet" "private" {
  filter {
    name   = "tag:Name" 
    values = ["Private-Subnet"] 
  }
}  

resource "aws_security_group" "ubuntusplunk_secgroup" {
  name = "ubuntusplunk_secgroup"
  description = "allow Splunk related traffic"
  vpc_id = data.aws_vpc.main.id

  ingress {
    description = "SSH from Bastion only"
    from_port = 22
    to_port = 22
    protocol = "tcp"
    cidr_blocks = ["10.0.1.176/32"]
  }

  ingress {
    description = "Splunk web UI"
    from_port = 8000
    to_port = 8000
    protocol = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  ingress {
    description = "Splunk indexing port for receiving logs"
    from_port = 9997
    to_port = 9997
    protocol  = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

    egress {
    from_port = 1024
    to_port = 65535
    protocol = "tcp"
  }

  egress {
    from_port = 80                                      
    to_port = 80
    protocol = "http"
  }

  egress {
    from_port = 443                                      
    to_port = 443
    protocol = "https"
  }

  tags = {
    Name = "ubuntusplunk_secgroup"
  }
}


```


---
### Creating the EC2 Instance That Will Become the Ubuntu Splunk Server 

Next, I asked Terraform to create the EC2 instance with an Ubuntu 22.04 LTS  image (`ami`), assign it to my private subnet, give it a private IP, associate a public IP, assign a key name for SSH login (via the `.tfvars` file), and assign it to the above security group. Additionally, I have allocated 50GiB of storage for the root hard drive as an SSD (`gp3`).

```tf
resource "aws_instance" "Ubuntu-Splunk-EC2" {
  ami = data.aws_ami.ubuntu_server.id.
  instance_type = "t3.medium"
  subnet_id = data.aws_subnet.private.id
  associate_public_ip_address = true
  key_name = var.key_name
  private_ip = "10.0.2.50"
  vpc_security_group_ids = [aws_security_group.ubuntusplunk_secgroup.id]

  tags = {
    Name = "Ubuntu-Splunk-EC2"
  }

  root_block_device {
    volume_size = 50
    volume_type = "gp3"
  }
}

```


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

Once the Ubuntu EC2 instance was created, I went ahead and SSH'd into it via my Bastion server (using the access key `.pem.` file) to verify that Splunk was installed and that the Splunk user was created as the owner of the Splunk file. However, Splunk did not download and install correctly due to a networking issue (NACL interference) and because I used the incorrect wget url link. So I had to manually download and install it::


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



## Now that I have created the Bastion and Domoain Controller servers, we are ready for [Phase 4 - Deploying the EC2 Splunk Server and Target Workstations](https://github.com/ChrisHerrera90/Complete-AWS-Cloud-Security-Architecture-Implementation-and-Testing-against-a-Simulated-Attack/blob/main/Phase%204%20-%20Deploying%20the%20EC2%20Splunk%20Server%20and%20Target%20Workstations/Phase%204%20README.md)
