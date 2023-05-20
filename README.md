# Terraform-VPC-creation-conditionally
This code shows a complete VPC creation and  how to create subnets based on the number of availability zones (AZs) available in a region using the count argument in Terraform as well as how to apply a condition for creating a NAT gateway and its associated Elastic IP (EIP) and for subnet assosiations. 

**Let's breakdown the code and explain how it works!**


> Variable definition for NAT resource


This variable definition can be used to configure whether or not to create a Network Address Translation (NAT) resource in your infrastructure provisioning code. By setting the default value to false, it means that the NAT resource will not be created by default unless the variable is explicitly set to true when using it.

```
variable "create_nat" { 
 
type = bool 
default = false 
}
```

> Creating Datasource

Retrieve the list of availability zones in a particular region.

```
data "aws_availability_zones" "azs" { 
 state = "available"
 }
 ```

> Defining Locals

A local value called "subnet_length" is defined using the Terraform "locals" block. The value of "subnet_length" is determined by the length of the list of availability zone names retrieved from the data


```
locals 
{ subnet_length = length(data.aws_availability_zones.azs.names) }
```

> Creating Elastic IP

The count argument is set to 1 if var.create_nat == true, indicating that one EIP should be created.

```
resource "aws_eip" "nat" {
  count = var.create_nat == true ? 1 : 0
  vpc   = true
}
```

> Creating NAT GW

NAT Gateway resource will be created when create_nat is set to true, and it will be associated with the specified EIP and subnet. If create_nat is set to false, no NAT Gateways will be created

```
resource "aws_nat_gateway" "nat" {
  count = var.create_nat == true ? 1 : 0
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.publicsubnet[0].id 
  depends_on = [aws_internet_gateway.igw]
  }
```

> Creating IGW

```
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}
```

> Creating public subnets

The code creates public subnets using the "aws_subnet" resource block. It uses the value of local.subnet_length (which represents the length of the availability zone names list) to determine the number of subnets to create.

```
resource "aws_subnet" "publicsubnet" { 
count = local.subnet_length 
vpc_id = aws_vpc.main.id 
cidr_block = cidrsubnet(var.vpc_cidr, 4, count.index) 
availability_zone = data.aws_availability_zones.azs.names[count.index] 
map_public_ip_on_launch = true 
tags = { 
Name = "${var.project_name}-${var.project_env}-public-${count.index + 1}" 
  } 
 } 
```
> Creating Private subnets

 Similarly, private subnets are created using the same "aws_subnet" resource block.

```
resource "aws_subnet" "privatesubnet" { 
count = local.subnet_length 
vpc_id = aws_vpc.main.id 
cidr_block = cidrsubnet(var.vpc_cidr, 4, "${count.index + local.subnet_length}") 
availability_zone = data.aws_availability_zones.azs.names[count.index] 
map_public_ip_on_launch = true 
tags = { 
Name = "${var.project_name}-${var.project_env}-private-${count.index + 1}" 
} 
   } 
 ```
 
 
