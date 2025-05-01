# Phase 3: Windows Active Directory Server EC2 Instance Deployment and Setup
In This phase, I will use Terraform to build out a Windows Server EC2 instance that will then be configured to be my Active Directory domain controller for other EC2 instances that will be built in later phases.

## 🎯 Main Target Goals for this Phase:
- Deploy a Windows EC2 domain controller in our private subnet with Terraform
- Configure this EC2 instance with sufficient CPU/memory to run Active Directory Domain services.
- Assign a static private IP to this EC2 instance
- Install and configure Active Directory services
- Create a new forest and domain
- Configure DNS and routing table settings for this EC2 domain access

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
I begin by creating the EC2 instance security group and naming it `windows_ad_secgroup` within my AWS environment. I also included `ingress` (inbound traffic) rules that allow open ports for RDP (so I can log into the EC2), DNS, and LDAP/Kerberos (for AD services to work with Linux) for communication within my private VPC network only. Finally, I included the `egress` (outbound traffic) rule to allow this security group to reach the internet and communicate with any protocol/port.

```tf
resource "aws_security_group "windows_ad_secgroup" {
  name = "windows_ad_secgroup"
  description = "allow AD-related traffic"
  vpc_id = aws_vpc.main.id

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

### Creating the EC2 Instance That Will Become the Windows AD Domain Controller
Next, I ask Terraform to create the EC2 instance with a Windows Server 2022 image (`ami`), assign it to my private subnet, give it a private IP, disallow a public IP, assign a key name for RDP login, assign it to the above security group, and even give it a powershell script that will automatically install Windows AD when it is enabled within the EC2 instance. Additionally, I have allocated 50GiB of storage for the root hard drive as an SSD (`gp3`).

```tf
resource "aws_instance" "windows_ad" {
  ami = "ami-0e999cbd62129e3b1"
  instance_type = "t3.medium"
  subnet_id = aws_subnet.private.id
  associate_public_ip_address = false
  key_name = var.key_name
  private_ip = "10.0.2.10"
  vpc_security_group_ids = [aws_security_group.windows_ad_secgroup.id]

  tags = {
    Name = "Windows-AD-EC2"
  }

  root_block_device {
    volume_size = 50
    volume_type = "gp3"
  }

  user_data = file("scripts/bootstrap_ad.ps1")
}

```

### Creating a 
Next, I need to create a variable key that can be used to log into the Windows EC2 server via RDP securely. I also created the `terraform.tfvars` file which holds the key values that will populate in the instance (`terraform.tfvars` file is redacted for security reasons)

```tf
variable "key_name" {
  description = "Name of the AWS key pair to access the Windows AD EC2 Instance"
  type = string
}
  
```










### XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

```tf
resource "aws_internet_gateway" "igw" {
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
  domain = "vpc"
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

  tags = {
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

resource "aws_route_table_association" "private" {
  subnet_id = aws_subnet.private.id
  route_table_id = aws_route_table.private.id
}

```

## </> Breakdown of `AWS Network Infrastructure Outputs.tf` for AWS Network Architecture Creation

 The output.tf file I constructed includes configuration details of all the following resources that will be created from my terraform script:
 - Main VPC instance
 - Public subnet
 - Private subnet
 - Internet Gateway
 - Elastic IP
 - NAT Gateway
 - Elastic IP
 - Public and Private Route Tables

```tf
output "vpc_id" {
  value = aws_vpc.main.id
  description = "The ID of the main VPC"
}

output "public_subnet_id" {
  value       = aws_subnet.public.id
  description = "The ID of the public subnet"
}

output "private_subnet_id" {
  value       = aws_subnet.private.id
  description = "The ID of the private subnet"
}

output "internet_gateway_id" {
  value       = aws_internet_gateway.igw.id
  description = "The ID of the Internet Gateway"
}

output "nat_gateway_id" {
  value       = aws_nat_gateway.nat.id
  description = "The ID of the NAT Gateway"
}

output "elastic_ip" {
  value       = aws_eip.nat.public_ip
  description = "The public IP address of the NAT Gateway"
}

output "public_route_table_id" {
  value       = aws_route_table.public.id
  description = "The ID of the public route table"
}

output "private_route_table_id" {
  value       = aws_route_table.private.id
  description = "The ID of the private route
}

```

---

## 👀 Executing and Verifying The Terraform Scripts Where Successful (AWS Dashboard)

After writing these scripts, I went ahead and ran them in my Visual Studio Terminal and checked if terraform validated my code with `terraform validate`:

<img width="811" alt="image" src="https://github.com/user-attachments/assets/70568ed8-9010-4bd1-8f10-e08d9ab472e2" />

As you can see, I had to revise two of the commands. Relatively simple fixes. Once remediated I ran the validation again and received a "success" message:

<img width="816" alt="image" src="https://github.com/user-attachments/assets/31cfe349-5293-4b5f-8a02-443119d11a17" />

With a successful validation, I initiated the `terraform plan` command to make sure that terraform understood exactly what I wanted to create within AWS:

```
PS C:\Users\Chris\OneDrive\Desktop\TF Phase 2 Building Core AWS Network Infrastructure with Terraform> terraform plan

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  + create

Terraform will perform the following actions:

  # aws_eip.nat will be created
  + resource "aws_eip" "nat" {
      + allocation_id        = (known after apply)
      + arn                  = (known after apply)
      + association_id       = (known after apply)
      + carrier_ip           = (known after apply)
      + customer_owned_ip    = (known after apply)
      + domain               = "vpc"
      + id                   = (known after apply)
      + instance             = (known after apply)
      + ipam_pool_id         = (known after apply)
      + network_border_group = (known after apply)
      + network_interface    = (known after apply)
      + private_dns          = (known after apply)
      + private_ip           = (known after apply)
      + ptr_record           = (known after apply)
      + public_dns           = (known after apply)
      + public_ip            = (known after apply)
      + public_ipv4_pool     = (known after apply)
      + tags_all             = (known after apply)
      + vpc                  = (known after apply)
    }

  # aws_internet_gateway.igw will be created
  + resource "aws_internet_gateway" "igw" {
      + arn      = (known after apply)
      + id       = (known after apply)
      + owner_id = (known after apply)
      + tags     = {
          + "Name" = "Main-IGW"
        }
      + tags_all = {
          + "Name" = "Main-IGW"
        }
      + vpc_id   = (known after apply)
    }

  # aws_nat_gateway.nat will be created
  + resource "aws_nat_gateway" "nat" {
      + allocation_id                      = (known after apply)
      + association_id                     = (known after apply)
      + connectivity_type                  = "public"
      + id                                 = (known after apply)
      + network_interface_id               = (known after apply)
      + private_ip                         = (known after apply)
      + public_ip                          = (known after apply)
      + secondary_private_ip_address_count = (known after apply)
      + secondary_private_ip_addresses     = (known after apply)
      + subnet_id                          = (known after apply)
      + tags                               = {
          + "Name" = "Main-NAT-GW"
        }
      + tags_all                           = {
          + "Name" = "Main-NAT-GW"
        }
    }

  # aws_route_table.private will be created
  + resource "aws_route_table" "private" {
      + arn              = (known after apply)
      + id               = (known after apply)
      + owner_id         = (known after apply)
      + propagating_vgws = (known after apply)
      + route            = [
          + {
              + cidr_block                 = "0.0.0.0/0"
              + gateway_id                 = (known after apply)
                # (11 unchanged attributes hidden)
            },
        ]
      + tags             = {
          + "Name" = "Private-RT"
        }
      + tags_all         = {
          + "Name" = "Private-RT"
        }
      + vpc_id           = (known after apply)
    }

  # aws_route_table.public will be created
  + resource "aws_route_table" "public" {
      + arn              = (known after apply)
      + id               = (known after apply)
      + owner_id         = (known after apply)
      + propagating_vgws = (known after apply)
      + route            = [
          + {
              + cidr_block                 = "0.0.0.0/0"
              + gateway_id                 = (known after apply)
                # (11 unchanged attributes hidden)
            },
        ]
      + tags             = {
          + "Name" = "Public-RT"
        }
      + tags_all         = {
          + "Name" = "Public-RT"
        }
      + vpc_id           = (known after apply)
    }

  # aws_route_table_association.private will be created
  + resource "aws_route_table_association" "private" {
      + id             = (known after apply)
      + route_table_id = (known after apply)
      + subnet_id      = (known after apply)
    }

  # aws_route_table_association.public will be created
  + resource "aws_route_table_association" "public" {
      + id             = (known after apply)
      + route_table_id = (known after apply)
      + subnet_id      = (known after apply)
    }

  # aws_subnet.private will be created
  + resource "aws_subnet" "private" {
      + arn                                            = (known after apply)
      + assign_ipv6_address_on_creation                = false
      + availability_zone                              = "us-east-1a"
      + availability_zone_id                           = (known after apply)
      + cidr_block                                     = "10.0.2.0/24"
      + enable_dns64                                   = false
      + enable_resource_name_dns_a_record_on_launch    = false
      + enable_resource_name_dns_aaaa_record_on_launch = false
      + id                                             = (known after apply)
      + ipv6_cidr_block_association_id                 = (known after apply)
      + ipv6_native                                    = false
      + map_public_ip_on_launch                        = false
      + owner_id                                       = (known after apply)
      + private_dns_hostname_type_on_launch            = (known after apply)
      + tags                                           = {
          + "Name" = "Private-Subnet"
        }
      + tags_all                                       = {
          + "Name" = "Private-Subnet"
        }
      + vpc_id                                         = (known after apply)
    }

  # aws_subnet.public will be created
  + resource "aws_subnet" "public" {
      + arn                                            = (known after apply)
      + assign_ipv6_address_on_creation                = false
      + availability_zone                              = "us-east-1a"
      + availability_zone_id                           = (known after apply)
      + cidr_block                                     = "10.0.1.0/24"
      + enable_dns64                                   = false
      + enable_resource_name_dns_a_record_on_launch    = false
      + enable_resource_name_dns_aaaa_record_on_launch = false
      + id                                             = (known after apply)
      + ipv6_cidr_block_association_id                 = (known after apply)
      + ipv6_native                                    = false
      + map_public_ip_on_launch                        = true
      + owner_id                                       = (known after apply)
      + private_dns_hostname_type_on_launch            = (known after apply)
      + tags                                           = {
          + "Name" = "Public-subnet"
        }
      + tags_all                                       = {
          + "Name" = "Public-subnet"
        }
      + vpc_id                                         = (known after apply)
    }

  # aws_vpc.main will be created
  + resource "aws_vpc" "main" {
      + arn                                  = (known after apply)
      + cidr_block                           = "10.0.0.0/16"
      + default_network_acl_id               = (known after apply)
      + default_route_table_id               = (known after apply)
      + default_security_group_id            = (known after apply)
      + dhcp_options_id                      = (known after apply)
      + enable_dns_hostnames                 = true
      + enable_dns_support                   = true
      + enable_network_address_usage_metrics = (known after apply)
      + id                                   = (known after apply)
      + instance_tenancy                     = "default"
      + ipv6_association_id                  = (known after apply)
      + ipv6_cidr_block                      = (known after apply)
      + ipv6_cidr_block_network_border_group = (known after apply)
      + main_route_table_id                  = (known after apply)
      + owner_id                             = (known after apply)
      + tags                                 = {
          + "Name" = "AWSsecuritylab-VPC"
        }
      + tags_all                             = {
          + "Name" = "AWSsecuritylab-VPC"
        }
    }

Plan: 10 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + elastic_ip             = (known after apply)
  + internet_gateway_id    = (known after apply)
  + nat_gateway_id         = (known after apply)
  + private_route_table_id = (known after apply)
  + private_subnet_id      = (known after apply)
  + public_route_table_id  = (known after apply)
  + public_subnet_id       = (known after apply)
  + vpc_id                 = (known after apply)

```
It checks out, the last step is to initiate the build by using the `terraform apply` command. Once initiated, we receive a message saying that our resources were successfully created:

<img width="834" alt="image" src="https://github.com/user-attachments/assets/82059833-2c89-4086-9b65-585ac4b4705c" />

If successful, the Terraform script execution should have resulted in the following AWS creations:
- VPC Instance
- Public Subnet
- Private Subnet
- Internet Gateway
- NAT gateway
- Elastic IP (for NAT)
- Public Routing Table
- Private Routing Table

Once the commands executed, I went to my AWS dashboard to confirm that all of the requested resources and configurations were implemented:

<img width="599" alt="image" src="https://github.com/user-attachments/assets/41505995-c6bf-4939-9ab4-ff77eeb48729" />
<img width="657" alt="image" src="https://github.com/user-attachments/assets/914be1e4-d9c2-4dde-87b8-e1f7319823d0" />
<img width="653" alt="image" src="https://github.com/user-attachments/assets/a6b1b048-c166-4da5-98d5-12e26d817841" />
<img width="659" alt="image" src="https://github.com/user-attachments/assets/eb8f11a2-d033-4aa9-9d1a-a2da6a9798e5" />
<img width="689" alt="image" src="https://github.com/user-attachments/assets/95533ea4-34da-4793-8f3e-c2e7f4957f71" />
<img width="619" alt="image" src="https://github.com/user-attachments/assets/e10e849a-5524-4773-8a99-75c2247ec604" />


## Now that I have created the networking infrastructure, we are ready for [Phase 3 - EC2 Instances Deployment and Setup](https://github.com/ChrisHerrera90/Complete-AWS-Cloud-Security-Architecture-Implementation-and-Testing-against-a-Simulated-Attack/tree/main/Phase%203%20-%20EC2%20Instances%20Deployment%20and%20Setup)

