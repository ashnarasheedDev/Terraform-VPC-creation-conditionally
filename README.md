# Terraform-VPC-creation-conditionally
This code shows a complete VPC creation and  how to create subnets based on the number of availability zones (AZs) available in a region using the count argument in Terraform as well as how to apply a condition for creating a NAT gateway and its associated Elastic IP (EIP) and for subnet assosiations. 

**Let's breakdown the code and explain how it works!**


> <b>Variable definition for NAT resource</b>


This variable definition can be used to configure whether or not to create a Network Address Translation (NAT) resource in your infrastructure provisioning code. By setting the default value to false, it means that the NAT resource will not be created by default unless the variable is explicitly set to true when using it.

```
variable "create_nat" { 
 
type = bool 
default = false 
}
```

> <b>Creating Datasource</b>

Retrieve the list of availability zones in a particular region.

```
data "aws_availability_zones" "azs" { 
 state = "available"
 }
 ```

> <b>Defining Locals</b>

A local value called "subnet_length" is defined using the Terraform "locals" block. The value of "subnet_length" is determined by the length of the list of availability zone names retrieved from the data


```
locals 
{ subnet_length = length(data.aws_availability_zones.azs.names) }
```

> <b>Creating Elastic IP</b>

The count argument is set to 1 if var.create_nat == true, indicating that one EIP should be created.

```
resource "aws_eip" "nat" {
  count = var.create_nat == true ? 1 : 0
  vpc   = true
}
```

> <b>Creating NAT GW</b>

NAT Gateway resource will be created when create_nat is set to true, and it will be associated with the specified EIP and subnet. If create_nat is set to false, no NAT Gateways will be created

```
resource "aws_nat_gateway" "nat" {
  count = var.create_nat == true ? 1 : 0
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.publicsubnet[0].id 
  depends_on = [aws_internet_gateway.igw]
  }
```

> <b>Creating IGW</b>

```
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
}
```

> <b>Creating public subnets</b>

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
> <b>Creating Private subnets</b>

 Similarly, private subnets are created using the same "aws_subnet" resource block.

```
resource "aws_subnet" "privatesubnet" { 
count = local.subnet_length 
vpc_id = aws_vpc.main.id 
cidr_block = cidrsubnet(var.vpc_cidr, 4, "${count.index + local.subnet_length}") 
availability_zone = data.aws_availability_zones.azs.names[count.index] 
map_public_ip_on_launch = false
tags = { 
Name = "${var.project_name}-${var.project_env}-private-${count.index + 1}" 
} 
   } 
 ```
 
 > <b>Creating Public route table</b>

```
resource "aws_route_table" "public" { 
vpc_id = aws_vpc.main.id 
 
route { 
cidr_block = "0.0.0.0/0" 
gateway_id = aws_internet_gateway.igw.id 
} 
  } 
```

 > <b>public rtb - publics subnets assosiation</b> 
 
This resource block will create multiple route table associations based on the value of local.subnet_length, associating each subnet specified by aws_subnet.publicsubnet with the route table specified by aws_route_table.public.
 
```
resource "aws_route_table_association" "public" { 
count = local.subnet_length 
subnet_id = aws_subnet.publicsubnet[count.index].id 
route_table_id = aws_route_table.public.id 
} 


 ```
 
 > <b>Creating Private rtb - NAT included</b>

This resource block creates an AWS route table with a single route that sends all traffic (0.0.0.0/0) to a NAT gateway specified by the aws_nat_gateway.nat[0] resource. The creation of the route table is conditional based on the value of the var.create_nat variable. If the variable is true, it will create the route table; otherwise, it will not create it.

```
resource "aws_route_table" "private_included_nat" {  
count = var.create_nat == true ? 1 : 0  
vpc_id = aws_vpc.main.id 
 
route { 
cidr_block = "0.0.0.0/0" 
nat_gateway_id = aws_nat_gateway.nat[0].id 
} 
 
 } 
```

> <b>Creating private rtb-excluding NAT</b>

This code is typically used to handle different routing configurations based on the presence or absence of a NAT gateway. The aws_route_table.private_excluded_nat resource block is used when the var.create_nat variable is set to false, indicating that there is no NAT gateway associated with this route table

 
```
resource "aws_route_table" "private_excluded_nat" {  
count = var.create_nat == false ? 1 : 0 
 vpc_id = aws_vpc.main.id  
 } 
 ```
 
> <b>Private rtb - private subnets assosiation with NAT</b> 

These resource blocks handle the association of subnets with the corresponding route tables based on the presence or absence of NAT, as determined by the var.create_nat variable.

```
resource "aws_route_table_association" "private_included_nat" { 

count = var.create_nat == true ? local.subnet_length : 0 
subnet_id = aws_subnet.privatesubnet[count.index].id 
route_table_id = aws_route_table.private_included_nat[0].id 

} 
 ```
 
 
> <b>Private rtb - Private subnets assosiation without NAT</b>
 
```
resource "aws_route_table_association" "private_excluded_nat" { 

count = var.create_nat == false ? local.subnet_length : 0 
subnet_id = aws_subnet.privatesubnet[count.index].id 
route_table_id = aws_route_table.private_excluded_nat[0].id 

}
```  

Overall, this code sets up a network infrastructure with public and private subnets, route tables, and their associations. Public subnets have internet connectivity, while private subnets can be configured to use a NAT gateway or exclude NAT depending on the value of the create_nat variable.<
