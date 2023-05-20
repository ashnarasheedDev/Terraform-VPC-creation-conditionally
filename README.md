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
