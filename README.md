# Terraform AWS Infrastructure Setup

This README provides a guide to creating a basic AWS infrastructure using Terraform. The setup includes a VPC, an Internet Gateway, a custom route table, a subnet, a security group, a network interface, an Elastic IP, and an Ubuntu server with Nginx installed.

## Steps

1. **Create VPC**: Define a Virtual Private Cloud (VPC) with a specified CIDR block.
2. **Create Internet Gateway**: Attach an Internet Gateway to the VPC to allow internet access.
3. **Create Custom Route Table**: Set up a route table and configure routes for both IPv4 and IPv6 traffic.
4. **Create a Subnet**: Define a subnet within the VPC and specify the CIDR block and availability zone.
5. **Associate Subnet with Route Table**: Associate the subnet with the custom route table created earlier.
6. **Create Security Group**: Define a security group to allow inbound traffic on ports 22 (SSH), 80 (HTTP), and 443 (HTTPS).
7. **Create Network Interface**: Create a network interface within the subnet and assign a private IP address.
8. **Assign Elastic IP**: Allocate and associate an Elastic IP address to the network interface.
9. **Create Ubuntu Server and Install Nginx**: Launch an Ubuntu instance, configure it with a network interface, and install Nginx using user data.

## Code

```hcl
provider "aws" {
  region = "us-east-1"
}

# Create vpc
resource "aws_vpc" "prod-vpc" {
  cidr_block = "10.0.0.0/16"
  tags = {
    name = "production"
  }
}

# Create Internet Gateway
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.prod-vpc.id
}

# Create Custom Route Table
resource "aws_route_table" "prod-route-table" {
  vpc_id = aws_vpc.prod-vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }

  route {
    ipv6_cidr_block = "::/0"
    gateway_id      = aws_internet_gateway.gw.id
  }

  tags = {
    Name = "prod"
  }
}

# Create a Subnet
resource "aws_subnet" "subnet-1" {
  vpc_id            = aws_vpc.prod-vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "prod-subnet"
  }
}

# Associate subnet with Route Table
resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.subnet-1.id
  route_table_id = aws_route_table.prod-route-table.id
}

# Create Security Group to allow port 22,80,443
resource "aws_security_group" "allow-web" {
  name        = "allow_web_traffic"
  description = "Allow web inbound traffic"
  vpc_id      = aws_vpc.prod-vpc.id

  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "allow_web"
  }
}

# Create a network interface with an IP in the subnet that was created
resource "aws_network_interface" "web-server-nic" {
  subnet_id       = aws_subnet.subnet-1.id
  private_ips     = ["10.0.1.50"]
  security_groups = [aws_security_group.allow-web.id]
}

# Assign an Elastic IP to the network interface created in step 7
resource "aws_eip" "one" {
  domain                    = "vpc"
  network_interface         = aws_network_interface.web-server-nic.id
  associate_with_private_ip = "10.0.1.50"
  depends_on                = [aws_internet_gateway.gw, aws_instance.web-server-instance]
}

# Create Ubuntu server and install Nginx
resource "aws_instance" "web-server-instance" {
  ami               = "ami-0e86e20dae9224db8"
  instance_type     = "t2.micro"
  availability_zone = "us-east-1a"
  key_name          = "main-key"

  network_interface {
    device_index         = 0
    network_interface_id = aws_network_interface.web-server-nic.id
  }

  user_data = <<-EOF
                #!/bin/bash
                sudo apt-get update -y
                sudo apt-get install -y nginx
                sudo systemctl start nginx
                sudo systemctl enable nginx
                EOF

  tags = {
    Name = "web-server"
  }
}

# Output
output "server_private_ip" {
  value = aws_instance.web-server-instance.private_ip
}

output "server_public_ip" {
  value = aws_eip.one.public_ip
}

output "server_id" {
  value = aws_instance.web-server-instance.id
}
