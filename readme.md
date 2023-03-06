https://gmusumeci.medium.com/how-to-deploy-ec2-instances-in-aws-using-terraform-45e304230262

How to Deploy EC2 Instances in AWS using Terraform
In this story, we will learn to deploy both Linux and Windows EC2 Instances in AWS using Terraform.

And also, we will learn how to deploy applications or configure settings at boot time (bootstrapping).

1. Requirements
To deploy a Virtual Machine in AWS, we will need:

AWS Credentials
Create an AWS Key Pair
Define AWS and Terraform Providers
Create a VPC, Subnet and other network components
Create Operating System Versions Variables
(Optional) Create a Bootstrapping script to install and/or configure applications
Create a Security Group
Create the EC2 Instance (Virtual Machine)
(Optional) Request a Public IP and attach to the EC2 Instance
Note: For clarity, we prefer to define separate files, however, you can put together the code, using the traditional main.tf, variables.tf and output.tf layout.

2. AWS Credentials
Before creating our AWS EC2 Instance, we will need AWS Credentials to execute our Terraform code.

The AWS provider offers a few options of providing credentials for authentication:

Static credentials
Environment variables
Shared credentials/configuration file
For this story, we will use static credentials. Please refer to the “How to create an IAM account and configure Terraform to use AWS static credentials?” story, if you need help to create the credentials.

Note: Using static credentials are great for learning and testing however hard-coded credentials are not recommended in production environments. Never push hard-coded credentials to code repositories.

3. AWS Key Pair
We will need an AWS Key Pair, consisting of a public key and a private key. The AWS Key Pair is a set of security credentials that we need to connect to an Amazon EC2 instance.

Amazon EC2 stores the public key on our instance, and we store the private key.

For Linux instances, the private key allows us to securely SSH into our instance. We can create the AWS Key Pair using the AWS Console, AWS CLI, or PowerShell. The instructions are at the “Amazon EC2 key pairs and Linux instances” official documentation.
For Windows instances, the private key allows us to obtain the administrator password and then log in the EC2 Instance using RDP. We can create the AWS Key Pair using the AWS Console, AWS CLI, or PowerShell. The instructions are at the “Amazon EC2 key pairs and Windows instances” official documentation.
A better way is using Terraform to create the AWS Key Pair. First, we will create a file called “key-pair-main.tf”, and we add the following code:

# Generates a secure private key and encodes it as PEM
resource "tls_private_key" "key_pair" {
  algorithm = "RSA"
  rsa_bits  = 4096
}
# Create the Key Pair
resource "aws_key_pair" "key_pair" {
  key_name   = "kopicloud-key-pair"  
  public_key = tls_private_key.key_pair.public_key_openssh
}
# Save file
resource "local_file" "ssh_key" {
  filename = "${aws_key_pair.key_pair.key_name}.pem"
  content  = tls_private_key.key_pair.private_key_pem
}
This code will generate an AWS Key Pair, and using the resource “local_file” will save the file to the folder where we run our Terraform code.

4. Defining AWS and Terraform Providers
First, we create a file called “provider-variables.tf”, used by the AWS authentication variables.

We will use an AWS Access Key, AWS Secret Key, and the AWS Region:

variable "aws_access_key" {
  type        = string
  description = "AWS access key"
}
variable "aws_secret_key" {
  type        = string
  description = "AWS secret key"
}
variable "aws_region" {
  type        = string
  description = "AWS region"
}
After that, we create the “provider-main.tf”, used to configure Terraform and the AWS provider:

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.0"
    }
  }
}
provider "aws" {
  access_key = var.aws_access_key
  secret_key = var.aws_secret_key
  region     = var.aws_region
}
Finally, we edit the file “terraform.tfvars” and add the AWS credential information (we will replace ‘complete-this’ strings with our values at run time):

aws_access_key = "complete-this"
aws_secret_key = "complete-this"
aws_region     = "eu-west-1"
5. Creating a Terraform file for the Network
In this step, we will create the file “network-variables.tf” to configure network variables and add the following code:

# AWS AZ
variable "aws_az" {
  type        = string
  description = "AWS AZ"
  default     = "eu-west-1c"
}
# VPC Variables
variable "vpc_cidr" {
  type        = string
  description = "CIDR for the VPC"
  default     = "10.1.64.0/18"
}
# Subnet Variables
variable "public_subnet_cidr" {
  type        = string
  description = "CIDR for the public subnet"
  default     = "10.1.64.0/24"
}
Then, we create the “network-main.tf” to configure the network and add the following code. This simple code will create a VPC, a public subnet, an internet gateway, and required routes.

# Create the VPC
resource "aws_vpc" "vpc" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
}
# Define the public subnet
resource "aws_subnet" "public-subnet" {
  vpc_id            = aws_vpc.vpc.id
  cidr_block        = var.public_subnet_cidr
  availability_zone = var.aws_az
}
# Define the internet gateway
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.vpc.id
}
# Define the public route table
resource "aws_route_table" "public-rt" {
  vpc_id = aws_vpc.vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }
}
# Assign the public route table to the public subnet
resource "aws_route_table_association" "public-rt-association" {
  subnet_id      = aws_subnet.public-subnet.id
  route_table_id = aws_route_table.public-rt.id
}
6. Creating a Terraform file for Operating System Versions Variables
We will create the “os-versions.tf” file, used to store variables for the different versions of operating systems.

6.1. Ubuntu Linux
# Get latest Ubuntu Linux Focal Fossa 20.04 AMI
data "aws_ami" "ubuntu-linux-2004" {
  most_recent = true
  owners      = ["099720109477"] # Canonical
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-*"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
6.2. Apple macOS
# Get latest Apple macOS Monterey 12 AMI
data "aws_ami" "mac-monterrey" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name = "name"
    values = ["amzn-ec2-macos-12*"]
  }
}
6.3. Amazon Linux
# Get latest Amazon Linux 2 AMI
data "aws_ami" "amazon-linux-2" {
  most_recent = true
  owners      = ["amazon"]  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm*"]
  }
}
6.4. CentOS
# Get latest CentOS 8 AMI
data "aws_ami" "centos-8" {
  most_recent = true
  owners      = ["125523088429"]
  filter {
      name   = "name"
      values = ["CentOS 8*"]
  }
  filter {
      name   = "architecture"
      values = ["x86_64"]
  }
  filter {
      name   = "root-device-type"
      values = ["ebs"]
  }
}
6.5. Red Hat Enterprise Linux (RHEL)
#  Get latest RHEL 8.5 AMI
data "aws_ami" "rhel_8_5" {
  most_recent = true
  owners      = ["309956199498"] // Red Hat's Account ID  
  
  filter {
    name   = "name"
    values = ["RHEL-8.5*"]
  }  
  filter {
    name   = "architecture"
    values = ["x86_64"]
  }  
  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }  
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
6.6. Debian Linux
# Get Latest Debian 11 Bullseye AMI
data "aws_ami" "debian-11" {
  most_recent = true
  owners      = ["136693071363"]  
  filter {
    name   = "name"
    values = ["debian-11-amd64-*"]
  }  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
6.7. Windows Server
# Get Latest Windows Server 2022 AMI
data "aws_ami" "windows-2022" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["Windows_Server-2022-English-Full-Base*"]
  }
}
7. (Optional) Creating a Bootstrapping Script
In this optional step, we are going to create a bootstrapping script to execute at the launch of the EC2 Instance. In the following example, we will install a web server. This extra step is useful to make sure the server is working properly.

Note: this step is optional. If we don’t want to deploy the web server, we will need to remove the “user_data” line from the “vm-main.tf” file and the HTTP rule in the security group.

7.1. Bootstrapping RHEL, CentOS & Amazon Linux Instances
We will use a simple Bash script, called “aws-user-data.sh” to install Apache Tomcat on the server.

#! /bin/bash
sudo yum update -y
sudo yum install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd
Then, we will use the “user_data” command in the resource “aws_instance” to execute the script.

7.2. Bootstrapping Debian & Ubuntu Linux Instances
We will use a simple Bash script, called aws-user-data.sh to install Apache Tomcat on the server.

#! /bin/bash
sudo apt-get update
sudo apt-get install -y apache2
sudo systemctl start apache2
sudo systemctl enable apache2
Then, we will use the “user_data” command in the resource “aws_instance” to execute the script.

7.2. Bootstrapping Windows Server Instances
We will use a simple PowerShell script to bootstrapping the EC2 instance. The script will rename the machine and install IIS (Internet Information Server) on the server.

We add the following code to the “vm-main.tf” file used to build the EC2 Instance.

# Bootstrapping PowerShell Script
data "template_file" "windows-userdata" {
  template = <<EOF
<powershell>
# Rename Machine
Rename-Computer -NewName "${var.windows_instance_name}" -Force;# Install IIS
Install-WindowsFeature -name Web-Server -IncludeManagementTools;# Restart machine
shutdown -r -t 10;
</powershell>
EOF
}
Then, we will use the “user_data” command in the resource “aws_instance” to call the data “template_file” resource.

9. Creating a Security Group
This code section will create the security group that allows incoming SSH (Linux), RDP (Windows Server) and HTTP connections.

We add the following code to the “vm-main.tf” file used to build the EC2 Instance.

# Define the security group for the EC2 Instance
resource "aws_security_group" "aws-vm-sg" {
  name        = "vm-sg"
  description = "Allow incoming connections"
  vpc_id      = aws_vpc.vpc.id  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow incoming HTTP connections"
  }
  ingress {
    from_port   = 3389
    to_port     = 3389
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow incoming RDP connections (Windows)"
  }  
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow incoming SSH connections (Linux)"
  }
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }  
  tags = {
    Name = "windows-sg"
  }
}
10. Creating a Terraform file for the VM Variables
Now we will create the “vm-variables.tf” file, used to store variables for the operating system.

variable "vm_instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t3.small"
}
variable "vm_associate_public_ip_address" {
  type        = bool
  description = "Associate a public IP address to the EC2 instance"
  default     = true
}
variable "vm_root_volume_size" {
  type        = number
  description = "Root Volume size of the EC2 Instance"
}
variable "vm_data_volume_size" {
  type        = number
  description = "Data volume size of the EC2 Instance"
}
variable "vm_root_volume_type" {
  type        = string
  description = "Root volume type of the EC2 Instance"
  default     = "gp2"
}
variable "vm_data_volume_type" {
  type        = string
  description = "Data volume type of the EC2 Instance"
  default     = "gp2"
}
11. Creating a Terraform file for the VM Main File
Finally, we create the file “vm-main.tf” to build the EC2 Instance. To update the version of operating system, update the ami line with a variable from the “os-versions.tf” file.

11.1. Linux EC2 Instance
This section of code will create a Linux EC2 Instance:

# Create EC2 Instance
resource "aws_instance" "vm-server" {
  ami                    = data.aws_ami.rhel_8_5.id
  instance_type          = var.vm_instance_type
  subnet_id              = aws_subnet.public-subnet.id
  vpc_security_group_ids = [aws_security_group.aws-vm-sg.id]
  source_dest_check      = false
  key_name               = aws_key_pair.key_pair.key_name
  associate_public_ip_address = var.vm_associate_public_ip_address
  
  user_data = file("aws-user-data.sh")
  
  # root disk
  root_block_device {
    volume_size           = var.vm_root_volume_size
    volume_type           = var.vm_root_volume_type
    delete_on_termination = true
    encrypted             = true
  }
  # extra disk
  ebs_block_device {
    device_name           = "/dev/xvda"
    volume_size           = var.vm_data_volume_size
    volume_type           = var.vm_data_volume_type
    encrypted             = true
    delete_on_termination = true
  }
  
  tags = {
    Name = "linux-server-vm"
  }
}
11.2. Windows Server EC2 Instance
This section of code will create a Windows Server EC2 Instance:

resource "aws_instance" "vm-server" {
  ami                    = data.aws_ami.windows-2022.id
  instance_type          = var.vm_instance_type
  subnet_id              = aws_subnet.public-subnet.id
  vpc_security_group_ids = [aws_security_group.aws-vm-sg.id]
  source_dest_check      = false
  key_name               = aws_key_pair.key_pair.key_name
  associate_public_ip_address = var.vm_associate_public_ip_address
  user_data = data.template_file.windows-userdata.rendered
  
  # root disk
  root_block_device {
    volume_size           = var.vm_root_volume_size
    volume_type           = var.vm_root_volume_type
    delete_on_termination = true
    encrypted             = true
  }  
  # extra disk
  ebs_block_device {
    device_name           = "/dev/xvda"
    volume_size           = var.vm_data_volume_size
    volume_type           = var.vm_data_volume_type
    encrypted             = true
    delete_on_termination = true
  }
  
  tags = {
    Name = "windows-server-vm"
  }
}
12. (Optional) Creating the Elastic IP (EIP)
In this optional step, we create an Elastic IP (EIP) and attach it to the EC2 instance.

We add the following code to the “vm-main.tf” file used to build the EC2 Instance.

# Create Elastic IP for the EC2 instance
resource "aws_eip" "vm-eip" {
  vpc  = true
  tags = {
    Name = "vm-eip"
  }
}
# Associate Elastic IP to the EC2 Instance
resource "aws_eip_association" "vm-eip-association" {
  instance_id   = aws_instance.vm-server.id
  allocation_id = aws_eip.vm-eip.id
}
13. Creating the Input Definition Variables File
In this step, we are going to create the input definition variables file “terraform.tfvars” and add the following code to it:

# Network
vpc_cidr           = "10.11.0.0/16"
public_subnet_cidr = "10.11.1.0/24"
# AWS Settings
aws_access_key = "complete-this"
aws_secret_key = "complete-this"
aws_region     = "eu-west-1"
# Virtual Machine Settings
vm_instance_name               = "kopisrv01"
vm_instance_type               = "t3.small"
vm_associate_public_ip_address = true
vm_root_volume_size            = 30
vm_root_volume_type            = "gp2"
vm_data_volume_size            = 10
vm_data_volume_type            = "gp2"
14. Creating the Output File
In the final step, we are going to create the output file “vm-output.tf” to show the result of the variables at the end of the execution of the Terraform code.

output "vm_server_instance_id" {
  value = aws_instance.vm-server.id
}
output "vm_server_instance_public_dns" {
  value = aws_instance.vm-server.public_dns
}
output "vm_server_instance_public_ip" {
  value = aws_instance.vm-server.public_ip
}
output "vm_server_instance_private_ip" {
  value = aws_instance.vm-server.private_ip
}
I wrote a few stories for different operating system, with more details and the full working code:

How to Deploy an Apple macOS EC2 Instance in AWS using Terraform
How to Deploy an Amazon Linux EC2 Instance in AWS using Terraform
How to Deploy a CentOS Linux EC2 Instance in AWS using Terraform
How to Deploy a Debian Linux EC2 Instance in AWS using Terraform
How to Deploy a Red Hat Enterprise Linux (RHEL) EC2 Instance in AWS using Terraform
How to Deploy an Ubuntu Linux EC2 Instance in AWS using Terraform
How to Deploy a Windows Server EC2 Instance in AWS using Terraform
And that’s all, folks. If you liked this story, please show your support by 👏 this story. Thank you for reading!
