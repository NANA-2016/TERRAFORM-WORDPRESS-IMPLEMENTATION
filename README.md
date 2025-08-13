# TERRAFORM-WORDPRESS-IMPLEMENTATION 

## ACCHIEVEMENTS


### NETWORKING
## CODES USED IN THE WHOLE PROJECT

## Provider.ft FILE

terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.55"
    }
  }
}

provider "aws" {
  region = var.aws_region
}


## Variables.tf file

variable "project"        { default = "wp-stack" }
variable "aws_region"     { default = "eu-north-1" }

# Networking
variable "vpc_cidr"       { default = "10.20.0.0/16" }
variable "public_subnets" { type = list(string) default = ["10.20.0.0/24", "10.20.1.0/24"] }
variable "private_subnets"{ type = list(string) default = ["10.20.10.0/24", "10.20.11.0/24"] }
variable "availability_zones" {
  type    = list(string)
  default = ["eu-north-1a", "eu-north-1b"]
}

# EC2 / Auto Scaling
variable "instance_type"  { default = "t3.small" }
variable "desired_capacity" { default = 2 }
variable "min_size"         { default = 2 }
variable "max_size"         { default = 4 }

# RDS
variable "db_name"        { default = "wordpress" }
variable "db_username"    { default = "wp_admin" }
variable "db_password"    { sensitive = true }
variable "db_instance_class" { default = "db.t4g.micro" } # cost-friendly starter
variable "db_allocated_storage" { default = 20 }

# EFS
variable "enable_efs_encryption" { default = true }

# ALB
variable "alb_idle_timeout" { default = 60 }

### vpc.tf file 

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = { Name = "${var.project}-vpc" }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags = { Name = "${var.project}-igw" }
}

resource "aws_subnet" "public" {
  for_each = toset(var.public_subnets)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value
  availability_zone       = var.availability_zones[index(var.public_subnets, each.value)]
  map_public_ip_on_launch = true
  tags = { Name = "${var.project}-public-${index(var.public_subnets, each.value)}" }
}

resource "aws_subnet" "private" {
  for_each = toset(var.private_subnets)
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value
  availability_zone = var.availability_zones[index(var.private_subnets, each.value)]
  tags = { Name = "${var.project}-private-${index(var.private_subnets, each.value)}" }
}

# Route tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = { Name = "${var.project}-public-rt" }
}

resource "aws_route_table_association" "public_assoc" {
  for_each       = aws_subnet.public
  subnet_id      = each.value.id
  route_table_id = aws_route_table.public.id
}

# NAT for private subnets
resource "aws_eip" "nat" {
  domain = "vpc"
  tags = { Name = "${var.project}-nat-eip" }
}

resource "aws_nat_gateway" "natgw" {
  allocation_id = aws_eip.nat.id
  subnet_id     = values(aws_subnet.public)[0].id
  tags = { Name = "${var.project}-natgw" }
  depends_on = [aws_internet_gateway.igw]
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.natgw.id
  }
  tags = { Name = "${var.project}-private-rt" }
}

resource "aws_route_table_association" "private_assoc" {
  for_each       = aws_subnet.private
  subnet_id      = each.value.id
  route_table_id = aws_route_table.private.id
}

Secrity_group.tf file

# ALB exposes HTTP (add 443 later if you bring ACM certs)
resource "aws_security_group" "alb_sg" {
  name   = "${var.project}-alb-sg"
  vpc_id = aws_vpc.main.id

  ingress { from_port = 80  to_port = 80  protocol = "tcp" cidr_blocks = ["0.0.0.0/0"] }
  egress  { from_port = 0   to_port = 0   protocol = "-1"  cidr_blocks = ["0.0.0.0/0"] }

  tags = { Name = "${var.project}-alb-sg" }
}

# EC2 instances only receive from ALB
resource "aws_security_group" "ec2_sg" {
  name   = "${var.project}-ec2-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description     = "HTTP from ALB"
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb_sg.id]
  }
  egress { from_port = 0 to_port = 0 protocol = "-1" cidr_blocks = ["0.0.0.0/0"] }

  tags = { Name = "${var.project}-ec2-sg" }
}

# EFS allows NFS from EC2
resource "aws_security_group" "efs_sg" {
  name   = "${var.project}-efs-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description     = "NFS"
    from_port       = 2049
    to_port         = 2049
    protocol        = "tcp"
    security_groups = [aws_security_group.ec2_sg.id]
  }
  egress { from_port = 0 to_port = 0 protocol = "-1" cidr_blocks = ["0.0.0.0/0"] }

  tags = { Name = "${var.project}-efs-sg" }
}

# RDS allows MySQL from EC2
resource "aws_security_group" "rds_sg" {
  name   = "${var.project}-rds-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    description     = "MySQL"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.ec2_sg.id]
  }
  egress { from_port = 0 to_port = 0 protocol = "-1" cidr_blocks = ["0.0.0.0/0"] }

  tags = { Name = "${var.project}-rds-sg" }
}

### rds.tf file

resource "aws_db_subnet_group" "db_subnets" {
  name       = "${var.project}-db-subnets"
  subnet_ids = [for s in aws_subnet.private : s.id]
  tags = { Name = "${var.project}-db-subnets" }
}

resource "aws_db_instance" "mysql" {
  identifier              = "${var.project}-mysql"
  engine                  = "mysql"
  engine_version          = "8.0"
  db_name                 = var.db_name
  username                = var.db_username
  password                = var.db_password
  instance_class          = var.db_instance_class
  allocated_storage       = var.db_allocated_storage
  storage_type            = "gp3"
  multi_az                = false
  db_subnet_group_name    = aws_db_subnet_group.db_subnets.name
  vpc_security_group_ids  = [aws_security_group.rds_sg.id]
  publicly_accessible     = false
  skip_final_snapshot     = true
  deletion_protection     = false
  backup_retention_period = 1
  tags = { Name = "${var.project}-mysql" }
}

output "rds_endpoint" {
  value = aws_db_instance.mysql.address
}


### EFS.tf. file

resource "aws_efs_file_system" "wp" {
  encrypted = var.enable_efs_encryption
  tags = { Name = "${var.project}-efs" }
}

# One mount target per AZ (use private subnets where EC2 lives)
resource "aws_efs_mount_target" "wp_mt" {
  for_each       = aws_subnet.private
  file_system_id = aws_efs_file_system.wp.id
  subnet_id      = each.value.id
  security_groups = [aws_security_group.efs_sg.id]
}

### iam.tf file

data "aws_iam_policy_document" "ec2_assume" {
  statement {
    actions = ["sts:AssumeRole"]
    principals { type = "Service" identifiers = ["ec2.amazonaws.com"] }
  }
}

resource "aws_iam_role" "ec2_role" {
  name               = "${var.project}-ec2-role"
  assume_role_policy = data.aws_iam_policy_document.ec2_assume.json
}

# Allow SSM and EFS mount helper
resource "aws_iam_role_policy_attachment" "ssm_core" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_iam_role_policy" "efs_client" {
  name = "${var.project}-efs-client"
  role = aws_iam_role.ec2_role.id
  policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect   = "Allow",
      Action   = ["elasticfilesystem:ClientMount", "elasticfilesystem:ClientWrite", "elasticfilesystem:ClientRootAccess"],
      Resource = "*"
    }]
  })
}

resource "aws_iam_instance_profile" "ec2_profile" {
  name = "${var.project}-ec2-profile"
  role = aws_iam_role.ec2_role.name
}

### userdate.tmpl.sh file

#!/bin/bash
set -euxo pipefail

# Vars injected by Terraform templatefile
EFS_ID="${efs_id}"
DB_HOST="${db_host}"
DB_NAME="${db_name}"
DB_USER="${db_user}"
DB_PASS="${db_pass}"

yum update -y
amazon-linux-extras enable php8.2 || true
yum install -y amazon-efs-utils nfs-utils httpd php php-mysqlnd php-gd php-xml php-mbstring php-zip mysql

systemctl enable --now httpd

# WordPress
cd /tmp
curl -L -o wordpress.tar.gz https://wordpress.org/latest.tar.gz
tar -xzf wordpress.tar.gz
rm -rf /var/www/html/*
mv wordpress/* /var/www/html/
chown -R apache:apache /var/www/html

# Mount EFS for wp-content
mkdir -p /var/www/html/wp-content
echo "${EFS_ID}:/ /var/www/html/wp-content efs _netdev,tls 0 0" >> /etc/fstab
mount -a

# Configure wp-config.php
cd /var/www/html
cp wp-config-sample.php wp-config.php
php -r "
\$f='wp-config.php';
\$c=file_get_contents(\$f);
\$c=str_replace('database_name_here', '${DB_NAME}', \$c);
\$c=str_replace('username_here', '${DB_USER}', \$c);
\$c=str_replace('password_here', '${DB_PASS}', \$c);
\$c=str_replace('localhost', '${DB_HOST}', \$c);
file_put_contents(\$f,\$c);
"

# Generate keys/salts
SALT=$(curl -s https://api.wordpress.org/secret-key/1.1/salt/)
php -r "
\$f='wp-config.php'; \$c=file_get_contents(\$f);
\$c=preg_replace('/define\\(\\s*\\'AUTH_KEY\\'.*/s','${SALT}',\$c,1);
file_put_contents(\$f,\$c);
"

# Ensure Apache perms
chown -R apache:apache /var/www/html
systemctl restart httpd

### alb_asg.tf file
# Get latest Amazon Linux 2 AMI (arm64 if using t4g; switch to x86_64 for t3)
data "aws_ssm_parameter" "amzn2_ami" {
  name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-6.1-x86_64"
}

resource "aws_lb" "app" {
  name               = "${var.project}-alb"
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = [for s in aws_subnet.public : s.id]
  idle_timeout       = var.alb_idle_timeout
  tags = { Name = "${var.project}-alb" }
}

resource "aws_lb_target_group" "tg" {
  name     = "${var.project}-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  health_check {
    path                = "/"
    matcher             = "200-399"
    healthy_threshold   = 2
    unhealthy_threshold = 5
    timeout             = 5
    interval            = 30
  }
  tags = { Name = "${var.project}-tg" }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.app.arn
  port              = 80
  protocol          = "HTTP"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tg.arn
  }
}

data "template_file" "userdata" {
  template = file("${path.module}/userdata.tmpl.sh")
  vars = {
    efs_id  = aws_efs_file_system.wp.id
    db_host = aws_db_instance.mysql.address
    db_name = var.db_name
    db_user = var.db_username
    db_pass = var.db_password
  }
}

resource "aws_launch_template" "lt" {
  name_prefix   = "${var.project}-lt-"
  image_id      = data.aws_ssm_parameter.amzn2_ami.value
  instance_type = var.instance_type
  iam_instance_profile { name = aws_iam_instance_profile.ec2_profile.name }
  vpc_security_group_ids = [aws_security_group.ec2_sg.id]
  user_data = base64encode(data.template_file.userdata.rendered)
  tag_specifications {
    resource_type = "instance"
    tags = { Name = "${var.project}-web" }
  }
}

resource "aws_autoscaling_group" "asg" {
  name                      = "${var.project}-asg"
  desired_capacity          = var.desired_capacity
  min_size                  = var.min_size
  max_size                  = var.max_size
  health_check_type         = "ELB"
  vpc_zone_identifier       = [for s in aws_subnet.private : s.id]
  target_group_arns         = [aws_lb_target_group.tg.arn]
  launch_template {
    id      = aws_launch_template.lt.id
    version = "$Latest"
  }
  lifecycle { create_before_destroy = true }
  tag {
    key                 = "Name"
    value               = "${var.project}-web"
    propagate_at_launch = true
  }
}

# Target tracking scaling policy on average CPU
resource "aws_autoscaling_policy" "cpu_tgt" {
  name                   = "${var.project}-cpu-tt"
  autoscaling_group_name = aws_autoscaling_group.asg.name
  policy_type            = "TargetTrackingScaling"
  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 50
  }
}

### output.tf

output "alb_dns_name" {
  value       = aws_lb.app.dns_name
  description = "Open this in your browser after apply."
}

output "efs_id" {
  value = aws_efs_file_system.wp.id
}


 #### ALL THESE FILES GIVE THE FOLLOWING RESULTS

1.VPC with public (ALB, NAT) and private subnets (EC2, RDS, EFS).

2. Route tables: public → IGW; private → NAT.

2. Security groups: ALB (80), EC2 (80 from ALB), EFS (2049 from EC2), RDS (3306 from EC2).

3. EFS for shared wp-content (mounted at boot via /etc/fstab).

4. RDS MySQL private; WordPress connects using vars.

5. ALB + Target Group + ASG (CPU target 50%), rolling-safe.

5. User data installs Apache/PHP, downloads WordPress, mounts EFS, writes wp-config.php with DB creds and salts.
 

Created a VPC with public and private subnet

Public subnet contains ALB,NAT GATEWAY with inbound rules and outbound to private subnets

<img width="1881" height="797" alt="image" src="https://github.com/user-attachments/assets/202898ee-da32-400a-a01c-18f4dc693c23" />

<img width="1892" height="791" alt="image" src="https://github.com/user-attachments/assets/b148bd4c-bd08-4a06-9922-6c713760ca02" />

Private subnet contains EC2 WEBTIER,RDS EFS . It can only be accessed through public subnet. with out bound via NAT GATEWAY only

<img width="1780" height="482" alt="image" src="https://github.com/user-attachments/assets/222301ae-4033-4e9b-abe1-57aba102730d" />

Below are the instances created:

<img width="1642" height="480" alt="image" src="https://github.com/user-attachments/assets/6c008d22-d617-4dbe-9746-ba345bc8a373" />

## AWS MySQL,RDS setup

Private subnets created which are not publicly available with subgroups restricting to EC2 subgroups only

## EFS setup

 the variables used to set it up are and was set up to allow access upload across instances

 <img width="1891" height="617" alt="image" src="https://github.com/user-attachments/assets/30585a53-f0f5-4ae5-a74d-7c1c525f0622" />


 ##ALB(Application Load Balancer)

 It is the only one that is exposed to the internet for security purposes as it fronts the enviroment.

 <img width="1871" height="457" alt="image" src="https://github.com/user-attachments/assets/6eab759f-16af-4fa9-b60b-ebaab4cc8d1b" />


 ## Autoscaling Group

 It ensure the whole system is secure and alsp reduces the risk of overload

 <img width="1883" height="362" alt="image" src="https://github.com/user-attachments/assets/4227dc3b-3fcd-43c2-bc93-be058baf3f6a" />


 The commands used all throught to allow setting up are 

 Terraform Init -To help initialize terraform

 Terraform Validate- To help validate the code 

 terraform plan -out tfplan -var 'db_password=CHANGE_ME_Strong!Passw0rd'

 teraform apply tfplan
