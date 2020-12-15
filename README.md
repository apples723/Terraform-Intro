# An Introduction to Terraform

A high level, yet detailed guide to Terraform. 

## Overview 

This is a guide that contains a selection of tidbits and information about Terraform. Included as well is a guide to create EC2 instances using the AWS EC2 Module. 

I put this together to act as a starting point to learning and utilizing Terraform. It highlights some of the things that stumped me when I first got started. 

It does assume you have a basic understanding of what and how Terraform works, Infrastructure as Code as well as AWS. I'd reccomend skimming [this](https://learn.hashicorp.com/tutorials/terraform/infrastructure-as-code?in=terraform/aws-get-started) official Terraform guide before getting started as well.

It's also good to note that most of the examples and recommendations are relevant to the way I use terraform at my job.

## Terraform Folder Structure

There are many ways to organize Terraform configuration files.  Here is an example of the current format my team uses:

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

The top-level folder is TF-Infrastructure.  This level holds all the configurations for my teams AWS accounts. Each enviornment has it's own AWS account.  It is a Git repository so we can version control our infrastructure. 

Every modlue should have at the very least the following:

- `main.tf` - This is the primary entry point for the module. It should contain the provider and backend configuration, at minimum.
- `variables.tf` - It should contain all variables used in the module..
- `outputs.tf` - It should contain all the outputs from the module that you may want to reference in a different module.

When creating an EC2 instance, I also use a file called `ec2.tf`.  This file contains the official Terraform AWS EC2 module.  While you can utilize the aws_instance resource to create the instance, for most of my use cases this is not necessary.

## Understanding Each File

Below is a high-level explanation for each configuration file highlighted above:

#### **`main.tf`** 
- This file contains the Terraform provider. The provider is responsible for understanding API interactions and exposing resources that configure resources in a specific infrastructure platform.  In my case I am using the `AWS` provider, but there are providers for about every cloud platform.
- Use the AWS provider:
````
provider "aws" { 
  provider = "aws" 
  region = "us-east-1" #tells terraform what region to use

}
````
- The provider block for AWS can also contain what account role you are using.  You could for example have a top level *Main Account*, *Sandbox Account*, or a *Sub Account*. You can have more than one provider as well, you use aliases, that you can call in resource blocks to tell Terraform to use a different account.
- To change the role you are using, add the `assume_role` argument to the provider block:

````
provider "aws" {
  provider = "aws"
  region   = "us-east-1"
  assume_role {
    role_arn = "full_arn_of_aws_account"
  }
}
````
- If you need a second account, you can declare another provider and assign it an alias.  Then inside of a resource, you can add the argument provider and call the provider's alias:

```
provider "aws" {
  provider = "aws"
  region   = "us-east-1"
  assume_role {
    role_arn = "second_account_full_arn" 
  }
  alias   = "second"
}

resource "aws_instance" "my_instance" {
  ami      = var.windows_ami
  type     = var.instance_type
  provider = aws.second
}
````
- This file also contains the configuration of our state file.  The state file is extremely important.  It explains to Terraform what my infrastructure looks like and compares the current configuration versus what's changed in the Terraform configuration files when running `terraform plan` and `terraform apply`.
- State files can be stored locally or in a backend system.  When working collaboratively with others on production or development infrastructure, never store your state file locally or in Git.
- I prefer to use a dedicated S3 bucket for all tf state files.  You can use the S3 backend by changing the backend like this:


````
terraform { 
  backend "s3" { 
    region         = "us-east-1" 
    bucket         = "bucket_name" 
    key            = "key_name" #change as necessary
    dynamodb_table = "terraform-locks" #prevents two people modifing the state at the same time
  }
}
````

#### **`variables.tf`**

- This file contains all variables used in the module.
- Variables are fairly self-explanatory and make writing reusable code easier.
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
- To use a variable, simply call it in an expression as var.<NAME>, where name matches the label given in the declaration block.  Here is an example:
  
````
resource "aws_instance" "my_instance" { 
  ami  = var.windows_ami
  type = var.type
}
````

#### **`outputs.tf`**

- This file contains all outputs values from a Terraform module.  You can then call from other modules using a data source to fetch for use elsewhere in a Terraform configuration.


# Utilizing the AWS EC2 Module

The AWS Terraform provider includes a module that makes creating EC2 instances extremely simple.  It's as simple as calling the Terraform AWS EC2 Module.


First declare a module block and set the source to the Terrform registry for the AWS EC2 Module.

```
module "my_instance_name" {
  source = "terraform-aws-modules/ec2-instance/aws" #path to aws ec instance on terraform registy
}
```

Next, pass arguments telling the module about the instance.  These are known as input variables.  The full list of input variables can be found [here](https://registry.terraform.io/modules/terraform-aws-modules/ec2-instance/aws/latest#inputs).

```
module "my_instance_name" {
  source = "terraform-aws-modules/ec2-instance/aws" #path to aws ec instance on terraform registy
  name   = var.name
  key    = var.key
  ami    = data.aws_ami.windows
  ...
}
```
