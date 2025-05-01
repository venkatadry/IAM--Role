# IAM--Role
Terraform VPC
EC2 with ASSUME role  s3

```#https://medium.com/@a-dem/create-a-private-public-vpc-in-aws-with-terraform-1d8e1b8118d2
###Provider
provider "aws" {
  region = "us-east-1"
}
##AWS VPC 
resource "aws_vpc" "example_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "VPC-Example"
  }
}

# Retrieve the default subnet ID in the default VPC

resource "aws_subnet" "public_subnet" {
  vpc_id     = aws_vpc.example_vpc.id
  cidr_block = "10.0.1.0/24"
  tags = {
    Name = "Vpc-Example-public-subnet"
  }
}
resource "aws_subnet" "private_subnet" {
  vpc_id     = aws_vpc.example_vpc.id
  cidr_block = "10.0.2.0/24"
  tags = {
    Name = "Vpc-Example-Private-subnet"
  }
}
resource "aws_internet_gateway" "example_igw" {
  vpc_id = aws_vpc.example_vpc.id
  tags = {
    Name = "Vpc-Example-IG"
  }
}
resource "aws_route_table" "example_rt" {
  vpc_id = aws_vpc.example_vpc.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.example_igw.id
  }
  tags = {
    Name = "Vpc-Example-rt-IG"
  }
}
resource "aws_route_table_association" "public_rt_association" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.example_rt.id
}

resource "aws_eip" "nat_eip" {
  vpc = true
}

resource "aws_nat_gateway" "example_nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = aws_subnet.public_subnet.id
  tags = {
    Name = "Vpc-Example-Nat-Gw"
  }
}
resource "aws_route_table" "private_rt" {
  vpc_id = aws_vpc.example_vpc.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.example_nat.id
  }
  tags = {
    Name = "Vpc-Example-Priate-Nat"
  }
}

resource "aws_route_table_association" "private_rt_association" {
  subnet_id      = aws_subnet.private_subnet.id
  route_table_id = aws_route_table.private_rt.id
}

resource "aws_security_group" "allow_web" {
  name        = "allow_web_traffic"
  description = "Allow SSH and HTTP inbound traffic"
  vpc_id      = aws_vpc.example_vpc.id
  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"] # Restrict to your IP in production
  }

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
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

# Launch EC2 instance in public subnet
resource "aws_instance" "web_server" {
  ami                         = "ami-0f88e80871fd81e91" # Amazon Linux 2 AMI
  instance_type               = "t2.micro"
  subnet_id                   = aws_subnet.public_subnet.id
  vpc_security_group_ids      = [aws_security_group.allow_web.id]
  iam_instance_profile        = aws_iam_instance_profile.alice_profile.name
  associate_public_ip_address = true
  key_name                    = "ec2-key" # Replace with your key pair name
  user_data                   = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get install -y apache2
              systemctl start apache2
              systemctl enable apache2
              echo "<h1>Hello World from $(hostname -f)</h1>" > /var/www/html/index.html
              EOF
  tags = {
    Name = "web-server"
  }
}


resource "aws_iam_user" "alice" {
  name = "Alice"
}

###IAM Assume Role
resource "aws_iam_role" "alice_role" {
  name = "AliceRole"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}


resource "aws_iam_instance_profile" "alice_profile" {
  name = "AliceProfile"
  role = aws_iam_role.alice_role.name
}

/*resource "aws_iam_user_instance_profile_attachment" "alice_profile_attachment" {
  user           = aws_iam_user.alice.name
  instance_profile = aws_iam_instance_profile.alice_profile.name
}*/


####IAM Policy to provide s3 Access
resource "aws_iam_policy" "allow_s3_access" {
  name        = "AllowS3Access"
  description = "Allow Alice to access specific S3 resources"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "s3:ListAllMyBuckets"
        ]
        Effect   = "Allow"
        Resource = ["*"]
      },
      {
        Action = [
          "s3:ListBucket",
          "s3:GetObject",
          "s3:PutObject"
        ]
        Effect   = "Allow"
        Resource = ["arn:aws:s3:::testabpvc", "arn:aws:s3:::testabpvc/*"]
      }

    ]
  })
}
###Attaching s3 access Policy to  the role alice_role
resource "aws_iam_role_policy_attachment" "attach_s3_access" {
  policy_arn = aws_iam_policy.allow_s3_access.arn
  role       = aws_iam_role.alice_role.name
}
###attaching IAM policy to IAM user
resource "aws_iam_user_policy_attachment" "alice_s3_access" {
  user       = aws_iam_user.alice.name
  policy_arn = aws_iam_policy.allow_s3_access.arn
}
```




```
[ec2-user@ip-10-0-1-20 ~]$ aws s3 ls testabpvc/ --recursive
2025-04-30 02:40:11       2981 2025-04-21-04-16-59-8251DA7FD85D304D.txt
2025-04-30 13:15:40       1674 sat
[ec2-user@ip-10-0-1-20 ~]$

[ec2-user@ip-10-0-1-20 ~]$ aws s3 ls test339943/ --recursive

An error occurred (AccessDenied) when calling the ListObjectsV2 operation: User: arn:aws:sts::920373005946:assumed-role/AliceRole/i-0a61c4b2b6c94c64d is not authorized to perform: s3:ListBucket on resource: "arn:aws:s3:::test339943" because no identity-based policy allows the s3:ListBucket action
[ec2-user@ip-10-0-1-20 ~]$
```


######
This Terraform block defines an AWS IAM Instance Profile named alice_profile. Here's a breakdown of what each line does:


```resource "aws_iam_instance_profile" "alice_profile" {
  name = "AliceProfile"
  role = aws_iam_role.alice_role.name
}
```
Explanation:
resource "aws_iam_instance_profile": This tells Terraform you're creating an IAM Instance Profile. This is required when assigning an IAM role to an EC2 instance.

"alice_profile": This is the name of the Terraform resource (not the actual AWS name). You use this name to reference the instance profile elsewhere in your Terraform code.

name = "AliceProfile": This is the actual name of the instance profile as it will appear in AWS.

role = aws_iam_role.alice_role.name: This line links the instance profile to an IAM Role you've defined elsewhere in your Terraform code (called alice_role). It uses that role's name as the associated role for the instance profile.

Why is this needed?
In AWS, EC2 instances can't directly assume an IAM role. Instead, they need to use an Instance Profile, which is a container for an IAM Role that can be attached to the instance.
