# Terraform-Intro
Contains a simple guide I wrote for my coworkers going over the basics of Terraform. It also demonstrates how to spin up an EC2 instance using the Terraform AWS EC2 Module.

## Overview 

The guide explains the basic setup of terraform module, and how to spin up an EC2 Instance using the Terraform AWS EC2 Module

## Terraform Folder Structure

There are many ways to organize terraform configuration files. Below is the way that I find works best for a 2 enviornment setup (prod & dev):

```bash
├── tf-infrastructure
│   ├── <ENVIORNMENT>
│   │   ├── <AWS REGION>
│   │   │   ├── <MODULE1>
│   │   │   │   ├── main.tf
│   │   │   │   └── other_config_file.tf
│   │   │   └── <MODULE2>
│   │   │       ├── main.tf
│   │   │       └── other_config_file.tf
│   ├── <ANOTHER ENVIORNMENT>
│   │   ├── <AWS REGION>
│   │   │   ├── <MODULE1>
│   │   │   │   ├── main.tf
│   │   │   │   ├── other_config_file.tf
│   │   │   └── <MODULE2>
│   │   │       ├── main.tf
│   │   │       └── other_config_file.tf
│   └── README.md
```

The top level folder is TF-Infrastructure, this holds all of the configurations for Prod and Dev, it's best to use Git repository so we can version control our infrastructure.

Every modlue should have at the very least the following:

- `main.tf` - This is the primary entry point for the module. It should contain at very least the provider and backend configuration. 
- `variables.tf` - Contains all variables used in the module
- `outputs.tf` - Contains all the outputs from the module that you may want to reference in a different module.

When creating an EC2 Instance we also have a file called `EC2.TF`. This file contains the official Terraform AWS EC2 module. You also can utlize the `aws_instance` resource to create an instance, it really depends on your use case. 

## Understanding Each File

Below is a high level explentation fo each configuation file higlighted above. 

#### **`main.tf`** 
- Contains the Terraform Provider. The provider is responsible for understanding API interactions and exposing resources that configure resources in a specific infrastructure platform. I will be using the `AWS` provider, but there are providers for about every cloud platform.  
- Using the AWS provider:
````
provider "aws" { 
  provider = "aws" 
  region = "us-east-1" #tells terraform what region to use

}
````
- The provider block for AWS also can contain what account role you are using. Depending on your use you may have more then one AWS account. You can have more then one provider as well with aliases that you can call in resource blocks to tell Terraform to use a different account.
- To change the role you are using add the `assume_role` argument to the provider block:
````
provider "aws" {
  provider = "aws"
  region   = "us-east-1"
  assume_role {
    role_arn = "full_arn_of_aws_account"
  }
}
````
- If you need a second account you can declare another provider and assign it an allias, then inside of a resourcce you can add the argument `provider` and add call the provider's allias: 
```
provider "aws" {
  provider = "aws"
  region   = "us-east-1"
  assume_role {
    role_arn = "second_account_full_arn" 
  }
  allias   = "second"
}

resource "aws_instance" "my_instance" {
  ami      = var.windows_ami
  type     = var.instance_type
  provider = aws.second
}
````
- It also contains the configuration of the state file. The state file extremly important. It explains to Terraform what our infrastructure looks like and compares the current configuration versus what's changed in the terraform config files when running `terraform plan` and `terraform apply`
- State files can be stored localy or in a backend system. When working colabrativily with others on production or dev infrastructure **never** store your state file locally or in git.
- I recommend utilizing a dedicated S3 bucket for all tf state files. You can use the S3 backend by changing the backend like this: 

````
terraform { 
  backend "s3" { 
    region         = "us-east-1" 
    bucket         = "tf-state" #bucket name
    key            = "<env>/<module_name>" #change as necessary
    dynamodb_table = "terraform-locks"
  }
}
````

#### **`variables.tf`**

- Contains all variables used in the module. 
- Variables are pretty self explanatory and make writing reusable code easier.
- You can declare a variable like this: 
````
variable "windows_ami" {
  type    = string #can be changed to different type of value (i.e nubmer, list, ect.)
  defualt = "ami-123456"
}
variable "instance_type" {
  type    = string
  defualt = "m5.large"
}
````
- To use a varaible simply call it in an expression as `var.<NAME>`, where name matches the label given in the decleration block. Here is an example: 
  
````
resource "aws_instance" "my_instance" { 
  ami  = var.windows_ami
  type = var.type
}
````

#### **`outputs.tf`**

- Contains all outputs values from a Terraform module that you can then call from other modules using a `data source` to fetch or computed for use elsewhere in a Terraform configuration


# Utilizing the AWS EC2 Module

The AWS Terraform provider includes a module that makes creating EC2 instances extremly simple. It's as easy as calling the Terraform AWS EC2 Module. 

First create a module block and set the source to the terrform registory for the AWS EC2 Module. 

```
module "my_instance_name" {
  source = "terraform-aws-modules/ec2-instance/aws" #path to aws ec instance on terraform registy
}
```

Next we need to pass arguments telling the module about our instance. These are know as input variables. The full list of input variables can be found [here](https://registry.terraform.io/modules/terraform-aws-modules/ec2-instance/aws/latest#inputs).

```
module "my_instance_name" {
  source = "terraform-aws-modules/ec2-instance/aws" #path to aws ec instance on terraform registy
  name   = var.name
  key    = var.key
  ami    = data.aws_ami.windows
}
```
