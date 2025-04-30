# Phase 2: Building Core AWS Network Infrastructure with Terraform
In This phase, I will use Terraform to build out the basic AWS network infrastructure needed for the requisite AWS services, Splunk integration, IAM configurations, logging, and more in the next phases.

## 🎯 Main Target Goals for this Phase:
Use Terraform to build the networking layer that will:
- Host Splunk (public subnet)
- Host Windows AD EC2 instance (private subnet)
- Enable internet access securely for both
- Set the foundation for IAM, security groups, logging, and hardening

---

# 🔧 Technology Utilized
- Terraform
- AWS VPC
- AWS Subnets
- AWS Internet Gateway (IGW)
- AWS NAT Gateway
- AWS Route Tables
- AWS Elastic IP (EIP)
- AWS Console (UI)

---

## 📽️ Video Walkthrough

------YOUTUBE VIDEO HERE------

---


# Table of Contents

- [Choosing the Provider Block (AWS)](#cis-control-1-inventory-and-control-of-assets)
- [Creating a new Virtual Private Cloud (VPC) Instance in AWS](#step-2-mock-meeting-policy-buy-in-stakeholders)
- [Creating Two Subnets within the VPC](#step-3-policy-finalization-and-senior-leadership-sign-off)
- [Creating The Internet Gateway (IGW) so Public Subnet can Access the Internet](#step-4-mock-meeting-initial-scan-permission-server-team)
- [Creating The NAT Gateway (IGW) and Elastic IP (EIP)](#step-5-initial-scan-of-server-team-assets)
- [Creating the Public and Private Route Tables](#step-7-distributing-remediations-to-remediation-teams)
- [asdasdsadsdsad](#step-7-distributing-remediations-to-remediation-teams)
- [asdasdsadsadsadasdsadsa](#step-7-distributing-remediations-to-remediation-teams)
- [Casdasdsadsadasdsa](#step-7-distributing-remediations-to-remediation-teams)
- [Csdsadasdsadsdsdass](#step-7-distributing-remediations-to-remediation-teams)

---

## </> Breakdown of `AWS Network Infrastructure.tf` for AWS Network Architecture Creation

### Choosing the Provider Block (AWS)
I begin the creation of this terraform script by choosing the cloud provider and the region in which I will deploy this script. In this case, it will be `AWS` in `us-east-1`:

```tf
provider "aws" {
  region = "us-east-1"
}
```

### Creating a new Virtual Private Cloud (VPC) Instance in AWS
Next, I include the creation of a new Virtual Private Cloud network labeled `AWS Security Lab-VPC` that will be used to connect all the resources within the lab. I also specified a Classless Inter-Domain Routing (CIDR) block of `10.0.0.0/16` to give me up to 65,536 designated IP addresses to use within this private network. Finally, I enabled DNS hostnames so that I can give a unique name to each EC2 instance within the network for easy identification:

```tf
Resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  enable_dns_hostnames = true
  tags = {
    Name = "AWSsecuritylab-VPC"
  }
}
```

### Creating Two Subnets within the VPC
Next, I needed to create two subnets within the VPC, one that is public facing (for Splunk, etc.) and one that is private (for Windows AD server, etc.). Since we have created an allotment of 65,536 IPs to use with our main VPC, we can now assign portions of this IP pool to our subnets. I do this by assigning a CIDR block of `10.0.1.0/24` and `10.0.2.0/24` to each subnet, so they each have a unique pool of 256 IPs to assign to assets and avoid IP overlap:

```tf
resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  availability zone = "us-east-1a"
  map_public_ip_on_launch = true
  tags = {
    Name = "Public-subnet"
  }
}

resource "aws_subnet" "private" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.2.0/24"
  availability_zone = "us-east-1a"
  tags = {
    Name = "Private-Subnet"
  }
}
  
```

### Creating The Internet Gateway (IGW) so Public Subnet can Access the Internet
Next, I needed to create an Internet Gateway (IGW) within my VPC so that any instances in my public subnet can access the public internet: 

```tf
resource = "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "Main-IGW"
  }
}
  
```

### Creating The NAT Gateway (IGW) and Elastic IP (EIP) so Private Subnet can Access the Internet Safely
Next, I needed to create a Network Address Translation (NAT) gateway and assign it a static public Elastic IP address. This NAT gateway will be used by my private subnet to access the public internet safely so it can be updated, patched, etc. The NAT gateway will be configured to prevent external services (SSH, RDP, etc.) from connecting to the instances found within my private subnet:

```tf
resource "aws_eip" "nat" {
  vpc = true
}

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat.id
  subnet_id = aws_subnet.public.id
  tags = {
    Name = "Main-NAT-GW"
  }
}
  
```

### Creating the Public and Private Route Tables so Private Subnet can Communicate with the Internet Through the  NAT Gateway
Next, I created the private and public route tables so that the private and public subnets understand where they are sending their network traffic/packets to when communicating over the internet. In this script, my public subnet is set up to send its traffic to the Internet Gateway (IGW) and my private subnet is set up to sends its traffic to the NAT Gateway. Once these two traffic sources reach their respective gateways, the gateways will then redirect their traffic to the public internet/network:

```tf
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "Public-RT"
  }
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_nat_gateway.nat.id
  }

  tags = }
    Name = "Private-RT"
  }
}

```

### Connecting the Public and Private Routing Tables to their Respective Subnets
Now that we have created the public and private routing tables for the NAT and IGW gateways, I now have to connect these routing tables to their respective public and private subnets. This will allow the subnets to follow their designated routing table rules and send their network traffic to the correct gateways for extra-net communication:

```tf
resource "aws_route_table_association" "public" {
  subnet_id = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_association" "private" {
  subnet_id = aws_subnet.private.id
  route_table_id = aws_route_table.private.id
}

```






## 👀 Verifying The Terraform Scripts Where Successful (AWS Dashboard)

List what should have happened as a results of these scripts:
- 

Screenshots of proof that they worked.


