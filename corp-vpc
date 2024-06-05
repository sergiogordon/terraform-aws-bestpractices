provider "aws" {
  region = "us-west-2"
}

variable "vpc_cidr" {
  default = "10.0.0.0/16"
}

variable "private_subnets" {
  default = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "public_subnets" {
  default = ["10.0.3.0/24", "10.0.4.0/24"]
}

variable "corp_cidr" {
  default = "192.168.1.0/24"  # Hypothetical CORP CIDR block
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "main_vpc"
  }
}

resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main_igw"
  }
}

resource "aws_subnet" "public" {
  count = length(var.public_subnets)

  vpc_id            = aws_vpc.main.id
  cidr_block        = element(var.public_subnets, count.index)
  map_public_ip_on_launch = true

  tags = {
    Name = "public_subnet_${count.index + 1}"
  }
}

resource "aws_subnet" "private" {
  count = length(var.private_subnets)

  vpc_id     = aws_vpc.main.id
  cidr_block = element(var.private_subnets, count.index)

  tags = {
    Name = "private_subnet_${count.index + 1}"
  }
}

resource "aws_nat_gateway" "nat" {
  count         = length(var.public_subnets)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = element(aws_subnet.public.*.id, count.index)

  tags = {
    Name = "main_nat_gateway_${count.index + 1}"
  }
}

resource "aws_eip" "nat" {
  count = length(var.public_subnets)

  vpc = true
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }

  tags = {
    Name = "public_rt"
  }
}

resource "aws_route_table_association" "public_assoc" {
  count          = length(var.public_subnets)
  subnet_id      = element(aws_subnet.public.*.id, count.index)
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "private_rt"
  }
}

resource "aws_route_table_association" "private_assoc" {
  count          = length(var.private_subnets)
  subnet_id      = element(aws_subnet.private.*.id, count.index)
  route_table_id = aws_route_table.private.id
}

resource "aws_route" "private_nat" {
  count = length(var.private_subnets)

  route_table_id         = aws_route_table.private.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = element(aws_nat_gateway.nat.*.id, count.index)
}

resource "aws_security_group" "allow_corp" {
  vpc_id = aws_vpc.main.id
  name   = "allow_corp_access"

  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = [var.corp_cidr]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "allow_corp_access"
  }
}

resource "aws_vpc_flow_log" "vpc_flow_log" {
  vpc_id = aws_vpc.main.id
  log_destination = aws_cloudwatch_log_group.vpc_flow_logs.arn
  traffic_type = "ALL"
}

resource "aws_cloudwatch_log_group" "vpc_flow_logs" {
  name = "/aws/vpc/flowlogs/main_vpc"
  retention_in_days = 90
}

resource "aws_network_acl" "private_nacl" {
  vpc_id = aws_vpc.main.id
  subnet_ids = aws_subnet.private.*.id

  ingress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = var.corp_cidr
    from_port  = 0
    to_port    = 65535
  }

  egress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 65535
  }

  tags = {
    Name = "private_nacl"
  }
}

resource "aws_network_acl" "public_nacl" {
  vpc_id = aws_vpc.main.id
  subnet_ids = aws_subnet.public.*.id

  ingress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = var.corp_cidr
    from_port  = 0
    to_port    = 65535
  }

  egress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 65535
  }

  tags = {
    Name = "public_nacl"
  }
}
