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
