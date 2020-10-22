# Hybrid_Task_4

## Problem Statement:
1. Write an Infrastructure as code using terraform, which automatically create a VPC.
2. In that VPC we have to create 2 subnets:
    a) public subnet [ Accessible for Public World! ]
    b) private subnet [ Restricted for Public World! ]
3. Create a public facing internet gateway for connect our VPC/Network to the internet world and attach this gateway to our VPC.
4. Create a routing table for Internet gateway so that instance can connect to outside world, update and associate it with public subnet.
5. Create a NAT gateway for connect our VPC/Network to the internet world and attach this gateway to our VPC in the public network
6. Update the routing table of the private subnet, so that to access the internet it uses the nat gateway created in the public subnet
7. Launch an ec2 instance which has Wordpress setup already having the security group allowing port 80 sothat our client can connect to our wordpress site. Also attach the key to instance for further login into it.
8. Launch an ec2 instance which has MYSQL setup already with security group allowing port 3306 in private subnet so that our wordpress vm can connect with the same. Also attach the key with the same.

## AWS VPC
Amazon Virtual Private Cloud (Amazon VPC) lets you provision a logically isolated section of the AWS Cloud where you can launch AWS resources in a virtual network that you define. You have complete control over your virtual networking environment, including selection of your own IP address range, creation of subnets, and configuration of route tables and network gateways. You can use both IPv4 and IPv6 in your VPC for secure and easy access to resources and applications.

## Subnet
Subnet is “part of the network”, in other words, part of entire availability zone. Each subnet must reside entirely within one Availability Zone and cannot span zones.

A subnet is a range of IP addresses in your VPC

Public Subnet :-If a subnet’s traffic is routed to an internet gateway, the subnet is known as a public subnet.
Private Subnet :-If a subnet doesn’t have a route to the internet gateway, the subnet is known as a private subnet.
## Route Table
A route table contains a set of rules, called routes, that are used to determine where network traffic from your subnet or gateway is directed.

## Internet Gateway
An internet gateway is a horizontally scaled, redundant, and highly available VPC component that allows communication between your VPC and the internet.

An internet gateway serves two purposes: to provide a target in your VPC route tables for internet-routable traffic, and to perform network address translation (NAT) for instances that have been assigned public IPv4 addresses.

An internet gateway supports IPv4 and IPv6 traffic.

## Nat Gateway
NAT Gateway, also known as Network Address Translation Gateway, is used to enable instances present in a private subnet to help connect to the internet or AWS services. In addition to this, the gateway makes sure that the internet doesn’t initiate a connection with the instances. NAT Gateway service is a fully managed service by Amazon, that doesn’t require any efforts from the administrator. Here we have to give a range of IP Address that is known as “CIDR” .

They don’t support IPV4 traffic. In the case of IPV4 traffic, an egress-only internet gateway needs to be used (which is another service).

# Implementation:
We will use that profile here:
```
provider "aws" {
  region     = "ap-south-1"
  profile = "aksharma"
}
```
## Step 1 : Write an Infrastructure as code using terraform, which automatically create a VPC.
Here we have to give a range of IP Address that is known as “CIDR” .
```
resource "aws_route_table" "route-table-bastion" {
  vpc_id = "${aws_vpc.task4.id}"
route {
    cidr_block = "0.0.0.0/0"
    gateway_id = "${aws_nat_gateway.gw.id}"
  }
tags ={
    Name = "nat-gateway-route-table"
  }
}
```


## Step 2: In that VPC we have to create 2 subnets:
1.) public subnet [ Accessible for Public World! ]

2.) private subnet [ Restricted for Public World! ]

public Subnet :- we have created this subnet in ap-south-1a zone with a range of ip addresses “192.168.0.0/24”
```
resource "aws_subnet" "private" {
   vpc_id = "${aws_vpc.task4.id}"
   cidr_block = "192.168.1.0/24"
   availability_zone = "ap-south-1b"
 }
resource "null_resource" "nulllocal1"  {
depends_on = [
    aws_vpc.task4,
	aws_subnet.public,
  ]
 }

resource "aws_subnet" "public" {
   vpc_id = "${aws_vpc.task4.id}"
   map_public_ip_on_launch = "true"
   cidr_block = "192.168.0.0/24"
   availability_zone = "ap-south-1a"
 }
resource "null_resource" "nulllocal2"  {
depends_on = [
    aws_vpc.task4,
  ]
 }
 ```

## Step 3 : Create a public facing internet gateway for connect our VPC/Network to the internet world and attach this gateway to our VPC.
```
resource "aws_internet_gateway" "test-env-gw" {
  vpc_id = "${aws_vpc.task4.id}"
tags ={
    Name = "test-env-gw"
  }
}
resource "null_resource" "nulllocal10"  {
depends_on = [
    aws_vpc.task4,
  ]
}
```

## Step 4 : Create a routing table for Internet gateway so that instance can connect to outside world, update and associate it with public subnet.
```
resource "aws_route_table" "route-table-test-env" {
  vpc_id = "${aws_vpc.task4.id}"
route {
    cidr_block = "0.0.0.0/0"
    gateway_id = "${aws_internet_gateway.test-env-gw.id}"
  }
tags ={
    Name = "test-env-route-table"
  }
}
resource "null_resource" "nulllocal11"  {
depends_on = [
    aws_internet_gateway.test-env-gw,
  ]
}

resource "aws_route_table_association" "subnet-association" {
  subnet_id      = "${aws_subnet.public.id}"
  route_table_id = "${aws_route_table.route-table-test-env.id}"
}
resource "null_resource" "nulllocal12"  {
depends_on = [
    aws_route_table.route-table-test-env,
    aws_internet_gateway.test-env-gw,
	aws_subnet.public,
	]
}
```

## Step 5 : Create a NAT gateway for connect our VPC/Network to the internet world and attach this gateway to our VPC in the public network.
```
resource "aws_eip" "example" {
  vpc = true
}
```
Create a NAT Gateway in the public subnet and create one route table for NAT Gateway and associate it with private subnet so that the instance running inside private subnet can go outside i.e. to the INTERNET but no outsider can come inside.
```
resource "aws_nat_gateway" "gw" {
  allocation_id = "${aws_eip.example.id}"
  subnet_id     = "${aws_subnet.public.id}"
}
resource "null_resource" "nulllocal3"  {
depends_on = [
    aws_eip.example,
  ]
 }

resource "aws_eip_association" "eip_assoc" {
  network_interface_id = "${aws_nat_gateway.gw.id}"
  allocation_id = "${aws_eip.example.id}"
}
resource "null_resource" "nulllocal4"  {
depends_on = [
    aws_nat_gateway.gw,
  ]
 }
 ```


## Step 6 : Update the routing table of the private subnet, so that to access the internet it uses the nat gateway created in the public subnet.
```
resource "aws_route_table" "route-table-bastion" {
  vpc_id = "${aws_vpc.task4.id}"
route {
    cidr_block = "0.0.0.0/0"
    gateway_id = "${aws_nat_gateway.gw.id}"
  }
tags ={
    Name = "nat-gateway-route-table"
  }
}
resource "null_resource" "nulllocal5"  {
depends_on = [
    aws_eip_association.eip_assoc,
  ]
 }

resource "aws_route_table_association" "subnet-private-association" {
  subnet_id      = "${aws_subnet.private.id}"
  route_table_id = "${aws_route_table.route-table-bastion.id}"
}
resource "null_resource" "nulllocal6"  {
depends_on = [
    aws_route_table.route-table-bastion,
 ]
 ```
 
## Step 7 : Launch an ec2 instance which has Wordpress setup already having the security group allowing port 80 so that our client can connect to our wordpress site. Also attach the key to instance for further login into it.
```
resource "tls_private_key" "taskkey" {
 algorithm = "RSA"
 rsa_bits = 4096
}
resource "aws_key_pair" "key" {
 key_name = "task-4-key"
 public_key = "${tls_private_key.taskkey.public_key_openssh}"
 depends_on = [
    tls_private_key.taskkey
]
}
```


- Security Group :- we have to allow traffic for Port No 80 and 22 .
```
resource "aws_security_group" "public-sg" {
  vpc_id = "${aws_vpc.task4.id}"
  name        = "task4-sg"
  ingress {
    description = "TCP"
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
    Name = "task4-sg"
  }
}
resource "null_resource" "nulllocal7"  {
depends_on = [
    aws_vpc.task4,
 ]
}
```

## Step 8 : Launch an ec2 instance which has MYSQL setup already with security group allowing port 3306 in private subnet so that our wordpress vm can connect with the same. Also attach the key with the same.
```
resource "aws_security_group" "private-sg" {
  vpc_id = "${aws_vpc.task4.id}"
  name        = "task4-sgp"
 ingress {
    description = "TCP"
    from_port   = 3306	
    to_port     = 3306
    protocol    = "tcp"
    security_groups = ["${aws_security_group.public-sg.id}","${aws_security_group.ssh.id}"]
}
  egress {
     from_port   = 0	
     to_port     = 0
     protocol    = "-1"
     cidr_blocks = ["0.0.0.0/0"]
}  
  tags = {
    Name = "task4-sgp"
  }
}
resource "null_resource" "nulllocal8"  {
depends_on = [
    aws_vpc.task4,
    aws_security_group.ssh,
	aws_security_group.public-sg,
  ]
}
```


### Code For MYSQL And WordPress Instances :
```
resource "aws_instance" "word" {
  ami           = "ami-000cbce3e1b899ebd"
  instance_type = "t2.micro"
  subnet_id = "${aws_subnet.public.id}"
  vpc_security_group_ids = ["${aws_security_group.public-sg.id}"]
  key_name = "task-4-key"
tags ={
    Name = "wordpress"
  }
}
resource "null_resource" "nulllocal13"  {
depends_on = [
    aws_route_table_association.subnet-association,
	local_file.key1,
  ]
}

resource "aws_instance" "bastion" {
  ami           = "ami-0732b62d310b80e97"
  instance_type = "t2.micro"
  subnet_id = "${aws_subnet.public.id}"
  vpc_security_group_ids = ["${aws_security_group.ssh.id}"]
  key_name = "task-4-key"
tags ={
    Name = "bastion"
  }
}
resource "null_resource" "nulllocal14"  {
depends_on = [
    local_file.key1,
	aws_route_table_association.subnet-private-association,
  ]
}

resource "aws_instance" "mysql" {
  ami           = "ami-08706cb5f68222d09"
  instance_type = "t2.micro"
  subnet_id = "${aws_subnet.private.id}"
  vpc_security_group_ids = ["${aws_security_group.private-sg.id}","${aws_security_group.ssh.id}"]
  key_name = "task-4-key"
tags ={
    Name = "mysql"
  }
}

resource "null_resource" "nulllocal15"  {
depends_on = [
    aws_route_table_association.subnet-association,
	local_file.key1,
  ]
}

resource "null_resource" "nulllocal16"  {
	provisioner "local-exec" {
	    command = "chrome  ${aws_instance.word.public_ip}"
  	}
}
resource "null_resource" "nulllocal17"  {
depends_on = [
    aws_instance.word,
	aws_instance.bastion,
	aws_instance.mysql,
  ]
}
```

### Then, Run the Terraform Code.
```
terraform init
```


### Now, we need is just to apply this code "terraform apply "
```
terraform apply -auto-approve
```

### - And at the end delete or destroy the complete process.
```
terraform destroy -auto-approve
```

## Thank You!!!
