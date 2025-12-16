Veriables*******

terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}

variable "region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "ap-south-1"
}

# Provide your existing EC2 key pair name (created separately or in console)
variablevariable "key_name" {
  description = "Existing AWS EC2 key pair name"
  type        = string
}

# Your public IP in CIDR (for SSH). Example: "203.0.113.10/32"
variable "ssh_ingress_cidr" {
  description = "CIDR allowed to SSH into web servers"
  type        = string
}

# RDS credentials (DO NOT commit real values to VCS)
variable "db_username" {
  description = "Master username for MySQL RDS"
  type        = string
  default     = "admin"
}

variable "db_password" {
  description = "Master password for MySQL RDS (store securely)"
  type        = string
  sensitive   = true
}

# Optional Route53
variable "hosted_zone_name" {
  description = "Existing Route53 hosted zone (e.g., example.com). Leave empty to skip record creation."
  type        = string
  default     = ""
}

# Record name: "" for root apex, or "www" / "app" etc.
variable "record_name" {
  description = "Record name for the ALB alias ('' for root, or subdomain like 'www')"
  type        = string
  default     = ""
}

locals {
  vpc_cidr = "10.0.0.0/17"

  # Subnet CIDRs
  public_subnet_1_cidr  = "10.0.0.0/21"   # ap-south-1a
  public_subnet_2_cidr  = "10.0.10.0/21"  # ap-south-1b
  private_subnet_1_cidr = "10.0.22.0/23"  # ap-south-1a
  private_subnet_2_cidr = "10.0.20.0/23"  # ap-south-1b

  az1 = "${var.region}a"
  az2 = "${var.region}b"

  Main.tf*******

  
############################
# VPC with DNS settings
############################
resource "aws_vpc" "two_tier" {
  cidr_block           = local.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "two-tier-vpc"
  }
}

############################
# Internet Gateway
############################
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.two_tier.id
  tags = {
    Name = "two-tier-igw"
  }
}

############################
# Subnets
############################
resource "aws_subnet" "public_1" {
  vpc_id                  = aws_vpc.two_tier.id
  cidr_block              = local.public_subnet_1_cidr
  availability_zone       = local.az1
  map_public_ip_on_launch = true
  tags = { Name = "public-subnet-1" }
}

resource "aws_subnet" "public_2" {
  vpc_id                  = aws_vpc.two_tier.id
  cidr_block              = local.public_subnet_2_cidr
  availability_zone       = local.az2
  map_public_ip_on_launch = true
  tags = { Name = "public-subnet-2" }
}

resource "aws_subnet" "private_1" {
  vpc_id            = aws_vpc.two_tier.id
  cidr_block        = local.private_subnet_1_cidr
  availability_zone = local.az1
  tags = { Name = "private-subnet-1" }
}

resource "aws_subnet" "private_2" {
  vpc_id            = aws_vpc.two_tier.id
  cidr_block        = local.private_subnet_2_cidr
  availability_zone = local.az2
  tags = { Name = "private-subnet-2" }
}

############################
# Route Tables
############################
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.two_tier.id
  tags   = { Name = "public-rt" }
}

# 0.0.0.0/0 -> IGW
resource "aws_route" "public_internet" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}

resource "aws_route_table_association" "public_assoc_1" {
  subnet_id      = aws_subnet.public_1.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "public_assoc_2" {
  subnet_id      = aws_subnet.public_2.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.two_tier.id
  tags   = { Name = "private-rt" }
}

resource "aws_route_table_association" "private_assoc_1" {
  subnet_id      = aws_subnet.private_1.id
  route_table_id = aws_route_table.private.id
}

resource "aws_route_table_association" "private_assoc_2" {
  subnet_id      = aws_subnet.private_2.id
  route_table_id = aws_route_table.private.id
}

############################
# Security Groups
############################

# ALB SG: Allow HTTP from anywhere
resource "aws_security_group" "alb_sg" {
  name        = "alb-sg"
  description = "ALB Security Group"
  vpc_id      = aws_vpc.two_tier.id

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "alb-sg" }
}

resource "aws_security_group_rule" "alb_ingress_http" {
  type              = "ingress"
  security_group_id = aws_security_group.alb_sg.id

  from_port   = 80
  to_port     = 80
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
  description = "Allow HTTP from internet to ALB"
}

# Web SG: Allow HTTP only from ALB SG, SSH from your IP
resource "aws_security_group" "web_sg" {
  name        = "web-sg"
  description = "Web Server SG"
  vpc_id      = aws_vpc.two_tier.id

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "web-sg" }
}

resource "aws_security_group_rule" "web_ingress_http_from_alb" {
  type                     = "ingress"
  security_group_id        = aws_security_group.web_sg.id
  source_security_group_id = aws_security_group.alb_sg.id

  from_port   = 80
  to_port     = 80
  protocol    = "tcp"
  description = "Allow HTTP from ALB only"
}

resource "aws_security_group_rule" "web_ingress_ssh" {
  type              = "ingress"
  security_group_id = aws_security_group.web_sg.id

  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = [var.ssh_ingress_cidr]
  description = "Allow SSH from admin IP"
}

# DB SG: Allow MySQL from web-sg only
resource "aws_security_group" "db_sg" {
  name        = "db-sg"
  description = "DB SG"
  vpc_id      = aws_vpc.two_tier.id

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "db-sg" }
}

resource "aws_security_group_rule" "db_ingress_mysql_from_web" {
  type                     = "ingress"
  security_group_id        = aws_security_group.db_sg.id
  source_security_group_id = aws_security_group.web_sg.id

  from_port   = 3306
  to_port     = 3306
  protocol    = "tcp"
  description = "Allow MySQL from web-sg only"
}

############################
# AMI lookup (Amazon Linux 2)
############################
data "aws_ami" "amazon_linux_2" {
  owners      = ["amazon"]
  most_recent = true

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

############################
# EC2 Web Servers
############################
resource "aws_instance" "web1" {
  ami                         = data.aws_ami.amazon_linux_2.id
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.public_1.id
  vpc_security_group_ids      = [aws_security_group.web_sg.id]
  key_name                    = var.key_name
  associate_public_ip_address = true

  user_data = templatefile("${path.module}/userdata-webserver.tpl", {
    message = "Web Server 1 Running"
  })

  tags = { Name = "web-server-1" }
}

resource "aws_instance" "web2" {
  ami                         = data.aws_ami.amazon_linux_2.id
  instance_type               = "t3.micro"
  subnet_id                   = aws_subnet.public_2.id
  vpc_security_group_ids      = [aws_security_group.web_sg.id]
  key_name                    = var.key_name
  associate_public_ip_address = true

  user_data = templatefile("${path.module}/userdata-webserver.tpl", {
    message = "Web Server 2 Running"
  })

  tags = { Name = "web-server-2" }
}

############################
# Target Group & Attachments
############################
resource "aws_lb_target_group" "web_tg" {
  name        = "web-tg"
  port        = 80
  protocol    = "HTTP"
  target_type = "instance"
  vpc_id      = aws_vpc.two_tier.id

  health_check {
    protocol            = "HTTP"
    path                = "/"
    matcher             = "200"
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 5
    interval            = 30
  }

  tags = { Name = "web-tg" }
}

resource "aws_lb_target_group_attachment" "web1_attach" {
  target_group_arn = aws_lb_target_group.web_tg.arn
  target_id        = aws_instance.web1.id
  port             = 80
}

resource "aws_lb_target_group_attachment" "web2_attach" {
  target_group_arn = aws_lb_target_group.web_tg.arn
  target_id        = aws_instance.web2.id
  port             = 80
}

############################
# Application Load Balancer & Listener
############################
resource "aws_lb" "web_alb" {
  name               = "web-alb"
  load_balancer_type = "application"
  internal           = false
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = [
    aws_subnet.public_1.id,
    aws_subnet.public_2.id
  ]

  tags = { Name = "web-alb" }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.web_alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web_tg.arn
  }
}

############################
# RDS Subnet Group & MySQL
############################
resource "aws_db_subnet_group" "db_subnet_group" {
  name       = "db-subnet-group"
  subnet_ids = [aws_subnet.private_1.id, aws_subnet.private_2.id]
  tags       = { Name = "db-subnet-group" }
  description = "Private DB subnets"
}

resource "aws_db_instance" "mysql" {
  identifier              = "new-mysql-db"
  engine                  = "mysql"
  instance_class          = "db.t3.micro"
  allocated_storage       = 20
  storage_type            = "gp2"
  username                = var.db_username
  password                = var.db_password
  db_subnet_group_name    = aws_db_subnet_group.db_subnet_group.name
  vpc_security_group_ids  = [aws_security_group.db_sg.id]
  publicly_accessible     = false
  multi_az                = false
  deletion_protection     = false
  backup_retention_period = 7

  # To fit Free Tier, omit fancy options; stick to minimal setup
  skip_final_snapshot = true

  tags = { Name = "new-mysql-db" }
}

############################
# Optional: Route 53 alias to ALB
############################
data "aws_route53_zone" "selected" {
  count = length(var.hosted_zone_name) > 0 ? 1 : 0
  name  = var.hosted_zone_name
}

resource "aws_route53_record" "alb_alias" {
  count   = length(var.hosted_zone_name) > 0 ? 1 : 0
  zone_id = data.aws_route53_zone.selected[0].zone_id

  name = var.record_name == "" ? data.aws_route53_zone.selected[0].name : "${var.record_name}.${data.aws_route53_zone.selected[0].name}"
  type = "A"

  alias {
    name                   = aws_lb.web_alb.dns_name
    zone_id                = aws_lb.web_alb.zone_id
    evaluate_target_health = false
  }
}

