# Phase 3: Windows Active Directory Server EC2 Instance Deployment and Setup
In This phase, I will use Terraform to build out a Windows Server EC2 instance that will then be configured to be my Active Directory domain controller for other EC2 instances that will be built in later phases.

## 🎯 Main Target Goals for this Phase:
- Create a security group for our Windows AD Server
- Deploy a Windows EC2 domain controller in our private subnet with Terraform
- Configure this EC2 instance with sufficient CPU/memory to run Active Directory Domain services.
- Deploy a EC2 Windows Bastion Host to RDP Into the AD EC2
- Install and configure Active Directory services
- Create a new forest and domain

---

# 🔧 Technology Utilized
- Terraform
- AWS EC2 with Windows Server (2022)
- Private Subnet
- Route Table with NAT Gateway access
- AWS Systems Manager (SSM) 
- RDP (Remote Desktop Protocol)
- Active Directory Domain Services (AD DS)
- 
---


# Table of Contents

- [Choosing the Provider Block (AWS)](#choosing-the-provider-block)
- [Creating a new Virtual Private Cloud (VPC) Instance in AWS](#creating-a-new-virtual-private-cloud-vpc-instance-in-aws)
- [Creating Two Subnets within the VPC](#creating-two-subnets-within-the-vpc)
- [Creating The Internet Gateway (IGW) so Public Subnet can Access the Internet](#creating-the-internet-gateway-igw-so-public-subnet-can-access-the-internet)
- [Creating The NAT Gateway (IGW) and Elastic IP (EIP)](#creating-the-nat-gateway-igw-and-elastic-ip-eip-so-private-subnet-can-access-the-internet-safely)
- [Creating the Public and Private Route Tables](#creating-the-public-and-private-route-tables-so-private-subnet-can-communicate-with-the-internet-through-the--nat-gateway)
- [Connecting the Public and Private Routing Tables to their Respective Subnets](#connecting-the-public-and-private-routing-tables-to-their-respective-subnets)
- [Breakdown of `Outputs.tf` for AWS Network Architecture Creation](#-breakdown-of-outputstf-for-aws-network-architecture-creation)
- [Executing and Verifying The Terraform Scripts Where Successful](#-executing-and-verifying-the-terraform-scripts-where-successful-aws-dashboard)

---

## ⭐ Step 1️: </> Breakdown of `Windows AD EC2 Creation.tf` for Windows AD Server EC2 Creation

### Creating the VPC Security Group for My Windows Domain Controller EC2

I begin by creating the EC2 instance security group and naming it `windows_ad_secgroup` within my AWS environment. I also included `ingress` (inbound traffic) rules that allow open ports for RDP (so I can log into the EC2), DNS, and LDAP/Kerberos (for AD services to work with Linux) for communication within my private VPC network only. Additionally, I included the `egress` (outbound traffic) rule to allow this security group to reach the internet and communicate with any protocol/port. Finally, I made sure to include some `data` blocks to ensure that any values such as the `ami`, `aws_vpc`, and `aws_subnet` was pulling values from what we setup in Phase 2.

```tf
provider "aws" {
  region = "us-east-1"
}

data "aws_ami" "windows_server" {
  most_recent = true
  owners      = ["801119661308"] 

  filter {
    name   = "name"
    values = ["Windows_Server-2025-English-Full-Base-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
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

resource "aws_security_group" "windows_ad_secgroup" {
  name = "windows_ad_secgroup"
  description = "allow AD-related traffic"
  vpc_id = data.aws_vpc.main.id

  ingress {
    description = "RDP"
    from_port = 3389
    to_port = 3389
    protocol = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "DNS"
    from_port = 53
    to_port = 53
    protocol = "udp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  ingress {
    description = "Kerberos"
    from_port = 88
    to_port = 88
    protocol  = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
  }

  tags = {
    Name = "windows_ad_secgroup"
  }
}


```


---
### Creating the EC2 Instance That Will Become the Windows AD Domain Controller

Next, I ask Terraform to create the EC2 instance with a Windows Server 2022 image (`ami`), assign it to my private subnet, give it a private IP, disallow a public IP, assign a key name for RDP login (via the `.tfvars` file), alter firewall settings to allow RDP, and assign it to the above security group. Additionally, I have allocated 50GiB of storage for the root hard drive as an SSD (`gp3`).

```tf
resource "aws_instance" "windows_ad" {
   ami = data.aws_ami.windows_server.id
  instance_type = "t3.medium"
  subnet_id = aws_subnet.private.id
  associate_public_ip_address = false
  key_name = var.key_name
  private_ip = "10.0.2.10"
  vpc_security_group_ids = [aws_security_group.windows_ad_secgroup.id]

user_data = <<-EOF
<powershell>
Set-ItemProperty -Path 'HKLM:\\System\\CurrentControlSet\\Control\\Terminal Server' -Name "fDenyTSConnections" -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
</powershell>
EOF

  tags = {
    Name = "Windows-AD-EC2"
  }

  root_block_device {
    volume_size = 50
    volume_type = "gp3"
  }
}

```

---
### Creating a Variable Key for RDP Access to Windows AD EC2

Next, I need to create a variable key that can be used to log into the Windows EC2 server via RDP securely. I also created the `terraform.tfvars` file which holds the key values that will populate in the instance (`terraform.tfvars` file is redacted for security reasons)

```tf
variable "key_name" {
  description = "Name of the AWS key pair to access the Windows AD EC2 Instance"
  type = string
}
  
```
---
### 👀 Executing and Verifying The Terraform Scripts Where Successful (AWS Dashboard)
After writing these scripts, I went ahead and ran them in my Visual Studio Terminal and checked if terraform validated my code with `terraform validate`:

![image](https://github.com/user-attachments/assets/cf80be60-301b-4938-a956-34b1f1faec3d)

With a successful validation, I went ahead and planned and executed the build by using the `terraform apply` command. Once initiated, we receive a message saying that our resources were successfully created:

![image](https://github.com/user-attachments/assets/ffdfcda9-9f96-47db-9840-6b9a3c90a980)

If successful, the Terraform script execution should have resulted in the following AWS creations:
- Security Group (windows_ad_secgroup)
- Windows Server EC2 Instance (Windows-AD-EC2)
- Private IP assigned to the EC2

Once the commands executed, I went to my AWS dashboard to confirm that all of the requested resources and configurations were implemented:

![image](https://github.com/user-attachments/assets/aa694225-3b56-4439-9fb4-5e62406bfa73)
![image](https://github.com/user-attachments/assets/eaf5a131-98ad-4cf9-baf4-ecc7bd50c985)



## ⭐ Step 2: Connecting to EC2 and Installing Active Directory Domain Services + Upgrading to Domain Controller

### Creating a Bastion Host (Jump Box) to Log into the Private EC2 via RDP
Since our `Windows-AD-EC2` EC2 instance does not have a public IP address (it is in our private subnet), I cannot just RDP into it locally to install Active Directory and upgrade it into my Domain Controller. I could just assign a temporary public IP, but in the spirit of maintaining security principles, I will instead create a Bastion Host (Jump Box) within my VPC public subnet and assign it a public IP. This Bastion Host (Windows EC2) will allow me to RDP into our `Windows-AD-EC2`'s private IP address. To automate this, I have executed the `Bastion Host Creation.tf` file. Here is the breakdown of that code:

---
### Creating The Bastion Host EC2 Security Group so I can RDP Into It
First, I needed to create the appropriate Security Group that will allow me to RDP into the Bastion Host. No other services will be activated. I also set up an `ingress` (inbound) rule that only allows my local IP address to connect via RDP for increased security.

```tf
provider "aws" {
  region = "us-east-1"
}

data "aws_ami" "windows_server" {
  most_recent = true
  owners      = ["801119661308"] 

  filter {
    name   = "name"
    values = ["Windows_Server-2025-English-Full-Base-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["AWSsecuritylab-VPC"] 
  }
}  

data "aws_subnet" "public" {
  filter {
    name   = "tag:Name"
    values = ["Public-subnet"] 
  }
}

resource "aws_security_group" "windows_bastion_secgroup" {
  name = "windows_bastion_secgroup"
  description = "allow bastion-related traffic"
  vpc_id = data.aws_vpc.main.id

  ingress {
    description = "RDP"
    from_port = 3389
    to_port = 3389
    protocol = "tcp"
    cidr_blocks = ["98.97.112.218/32"]
  }

  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
  }

  tags = {
    Name = "Bastion-Host-Windows"
  }
}

```

---
### Creating the EC2 Instance That Will Become the Bastion Host (Jump Host)

Next, I ask Terraform to create the EC2 instance with a Windows Server 2025 image (`ami`), assign it to my PUBLIC subnet, give it a public IP, assign a key name for RDP login (via the `.tfvars` file), and assign it to the above security group. Additionally, I have allocated 30GiB of storage for the root hard drive as an SSD (`gp3`).

```tf
variable "key_name" {
  description = "The name of the existing EC2 Key Pair to use for Bastion Host access"
  type        = string
}

resource "aws_instance" "windows_bastion" {
  ami = data.aws_ami.windows_server.id
  instance_type = "t3.medium"
  subnet_id = data.aws_subnet.public.id
  associate_public_ip_address = true
  key_name = var.key_name
  vpc_security_group_ids = [aws_security_group.windows_bastion_secgroup.id]

  tags = {
    Name = "Windows-Bastion-EC2"
  }

  root_block_device {
    volume_size = 30
    volume_type = "gp3"
  }
}

```

After validating and applying the Terraform commands, I went into the AWS dashboard to confirm the creation of the `windows_bastion` EC2 and `windows_bastion_secgroup` security group:

![image](https://github.com/user-attachments/assets/1cecff15-af71-40e3-bb42-bac0f87576cf)


---
### RDP Into Windows AD EC2 and Installing Active Directory and Upgrading it to the Domain Controller

Now that we have our Bastion Host setup, we are going to RPD into it via the AWS RDP Client using our key pair from the `.pem` file we created.

![image](https://github.com/user-attachments/assets/9af33fae-0a81-4e18-938e-756408908246)

Once we are in our Bsation Host EC2, we will then use it to RDP into our `Windows-AD-EC2` so we can install and setup Active Directory within it:

![image](https://github.com/user-attachments/assets/4199d114-e05a-474d-9947-35836ef33eb4)



---
### Configuring Active Directory Users and Groups for the Lab








## Now that I have created the networking infrastructure, we are ready for [Phase 3 - EC2 Instances Deployment and Setup](https://github.com/ChrisHerrera90/Complete-AWS-Cloud-Security-Architecture-Implementation-and-Testing-against-a-Simulated-Attack/tree/main/Phase%203%20-%20EC2%20Instances%20Deployment%20and%20Setup)

