---
title: "Stop Manually Clicking! How Terraform Made My AWS & Jenkins Learning Journey Effortless"
description: Set up AWS EC2 with Jenkins using Terraform
author: Vaibhav Gagneja
date: 2026-02-06 17:44:46 +0000
categories: [DevOps, Terraform]
tags: [terraform, aws, jenkins, infrastructure-as-code, automation]
image:
  path: https://cdn.hashnode.com/res/hashnode/image/upload/v1770399094548/fecac1bb-e823-4177-8419-b165cf28d812.jpeg
---

I recently resumed my journey learning AWS, and I hit a realization that changed everything.

In the past, whenever I wanted to practice with a tool like Jenkins, I would log into the AWS Console, click through the EC2 wizard, manually select security groups, SSH in, type command after command to install dependencies, and hope I didn't miss a step.

Then, when I was done, I had to remember to go back and delete everything. If I forgot? I paid for it—literally.

I decided to switch gears and use **Terraform** to automate my lab setup. The result? It is shockingly easy to provision and deprovision infrastructure. I can spin up a fully configured Jenkins server in two minutes, practice, and destroy it immediately after.

Here is how I did it, and how you can do it too.

## The Goal

We want to create a "One-Click" Jenkins server that:

1. Launches an EC2 instance in a default VPC.
2. Opens Port 8080 (for the web UI) and Port 22 (for SSH).
3. Automatically installs Java and Jenkins.
4. Prints the URL and the initial admin password.

## Prerequisite: The Setup

Before running the code, make sure you have:

* Terraform installed.
* AWS CLI configured with your credentials.
* An SSH Key Pair created in AWS named `ec2withJenkinsPair` (and the `.pem` file saved locally in your project folder).

## Step 1: The Infrastructure (main.tf)

This Terraform file handles the heavy lifting. It uses a `null_resource` with a `remote-exec` provisioner to move our installation script onto the server and run it.

```hcl
provider "aws" {
  region    = "us-east-1"
}

# 1. Network Setup: Using Default VPC and Subnet for simplicity
resource "aws_default_vpc" "default_vpc" {
  tags = {
    Name = "default vpc"
  }
}

data "aws_availability_zones" "available_zones" {}

resource "aws_default_subnet" "default_az1" {
  availability_zone = data.aws_availability_zones.available_zones.names[0]
  tags = {
    Name = "default subnet"
  }
}

# 2. Security Group: Opening 8080 (Jenkins) and 22 (SSH)
resource "aws_security_group" "ec2_security_group" {
  name        = "ec2 security group"
  description = "allow access on ports 8080 and 22"
  vpc_id      = aws_default_vpc.default_vpc.id

  ingress {
    description = "http proxy access"
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "ssh access"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = -1
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "jenkins server security group"
  }
}

# 3. Get the latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "owner-alias"
    values = ["amazon"]
  }

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm*"]
  }
}

# 4. Launch the EC2 Instance
resource "aws_instance" "ec2_instance" {
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = "t3.micro"
  subnet_id              = aws_default_subnet.default_az1.id
  vpc_security_group_ids = [aws_security_group.ec2_security_group.id]
  key_name               = "youKeyPair"

  tags = {
    Name = "jenkins-server"
  }
}

# 5. The Magic: Connect via SSH and install Jenkins
resource "null_resource" "name" {

  # Establish connection
  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file("./youKeyPair.pem")
    host        = aws_instance.ec2_instance.public_ip
  }

  # Copy the script to the remote server
  provisioner "file" {
    source      = "install_jenkins.sh"
    destination = "/tmp/install_jenkins.sh"
  }

  # Execute the script
  provisioner "remote-exec" {
    inline = [
      "sudo chmod +x /tmp/install_jenkins.sh",
      "sh /tmp/install_jenkins.sh"
    ]
  }

  depends_on = [aws_instance.ec2_instance]
}

# 6. Output the URL for easy access
output "website_url" {
  value = join ("", ["http://", aws_instance.ec2_instance.public_dns, ":", "8080"])
}
```

## Step 2: The Installation Script (install_jenkins.sh)

Instead of typing commands manually, we save them in a bash script. This installs Java 17 (required for newer Jenkins versions) and the Jenkins service itself.

```bash
#!/bin/bash

# Update the system
sudo yum update -y

# Install Java 17 (Amazon Corretto)
sudo yum install java-17-amazon-corretto -y

# Add the Jenkins Repo
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/rpm-stable/jenkins.repo

# Import the key (Notice the 2026 key for future-proofing)
sudo rpm --import https://pkg.jenkins.io/rpm-stable/jenkins.io-2026.key

# Upgrade and Install Jenkins
sudo yum upgrade -y
sudo yum install jenkins -y

# Enable and Start Jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins

# Check status
sudo systemctl status jenkins

# PRINT THE ADMIN PASSWORD to the logs
echo "Waiting for Jenkins to generate password..."
sleep 10 # give it a moment to generate the file
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

## How to Run It

This is where the magic happens. Open your terminal in the folder containing these two files and your `.pem` key.

**1. Initialize Terraform:**

```bash
terraform init
```

**2. Provision the Infrastructure:**

```bash
terraform apply --auto-approve
```

Sit back and watch. Terraform will create the network, launch the instance, and stream the logs of the installation script right to your terminal.

**3. Access Jenkins:**

At the very end of the logs, Terraform will output the `website_url`. Copy that into your browser.

Also, look slightly up in your terminal logs. Because we added `sudo cat /var/lib/jenkins/secrets/initialAdminPassword` to the script, **your login password is printed right there in the terminal!** No need to SSH in to find it.

## The Best Part: Deprovisioning

When you are done studying for the day, you don't need to hunt for instances or worry about stray volumes costing you money. Just run:

```bash
terraform destroy --auto-approve
```

Everything is wiped clean.

## Conclusion

Learning AWS and DevOps tools like Jenkins doesn't have to be a manual chore. By codifying the infrastructure, you save time, save money, and get to practice "Infrastructure as Code" concepts while you learn.

Shout out to this video: [https://youtu.be/9XrYwfIWDL0](https://youtu.be/9XrYwfIWDL0?si=gpR9uPrPwCi4_Bhm)

Give this setup a try for your next lab session!
