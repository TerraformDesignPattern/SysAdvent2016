## Introduction

This article was originally published during [SysAdvent 2016](http://sysadvent.blogspot.com/2016/12/day-14-terraform-deployment-strategy.html).

[HashiCorp's](https://www.hashicorp.com/) infrastructure management tool, [Terraform](https://www.terraform.io/), is no doubt very flexible and powerful.  The question is, how do we write Terraform code and construct our infrastructure in a reproducible fashion that makes sense? How can we keep code [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself), segment state, and reduce the risk of making changes to our service/stack/infrastructure?

This post describes a design pattern to help answer the previous questions. This article is divided into two sections, with the first section describing and defining the design pattern with a _Deployment Example_. The second part uses a [multi-repository GitHub organization](https://github.com/TerraformDesignPattern) to create a _Real World Example_ of the design pattern.

This post assumes you understand or are familiar with AWS and basic Terraform concepts such as <a href="https://www.terraform.io/docs/commands/index.html">CLI Commands</a>, <a href="https://www.terraform.io/docs/configuration/providers.html">Providers</a>, <a href="https://www.terraform.io/docs/providers/aws/index.html">AWS Provider</a>, <a href="https://www.terraform.io/docs/state/index.html">Remote State</a>, <a href="https://www.terraform.io/docs/providers/terraform/d/remote_state.html">Remote State Data Sources</a>, and <a href="https://www.terraform.io/docs/modules/index.html">Modules</a>.

### Modules

Essentially, any directory with one or more `.tf` files can be used as or considered a Terraform module. I am going to be creating a couple of _module types_ and giving them names for reference. The first module type constructs all of the resources a service will need to be operational such as EC2 instances, S3 bucket, etc. The remaining module types will instantiate the aforementioned module type.

The first module type is a _service module._ A service module can be thought of as a reuseable library that deploys a single service's infrastructure. _Service modules_ are the brains and contain the logic to create Terraform resources. They are the _"how"_ we build our infrastructure. 

The other module types are _environment modules._ We will run our Terraform commands within this module type. The environment modules all live within a single repository, as compared to service modules, which live in individual repositories or alongside the code of the service they build infrastructure for. This is _"where"_ our infrastructure will be built.

## Deployment Example

I am going to start by describing how we would deploy a service and then deconstruct the concepts as we move through the deployment.

### Running Terraform

As mentioned earlier, the [environments repository](https://github.com/TerraformDesignPattern/environments) is where we actually run the Terraform command to instantiate a service module. I've written a [Bash wrapper](https://github.com/TerraformDesignPattern/environments/blob/master/templates/service-remote.sh) to manage the service's remote state configuration and ensure we always have to latest modules. 

So instead of running `terraform apply` we will run `./remote.sh apply` 

The `apply` argument will set and get our remote state configuration, run a `terraform get -update` and then run `terraform apply`

### Environment Module Example

#### Directory Structure

The environment module's respository contains a strict directory hierarchy:

```
production-account (aws-account)
|__ us-east-1 (aws-region)
    |__ production-us-east-1-vpc (vpc)
        |__ production (environment)
            |__ ssh-bastion (service) 
               |__ remote.sh <~~~~~ YOU ARE HERE
```

##### Dynamic State Management

The directory structure is one of the cornerstones of this design as it enables us to dynamically generate the S3 key for a service's state file. Within the remote.sh script (shown below) we parse the directory structure and then set/get our remote state. 

A symlink of `/templates/service-remote.sh` to `remote.sh` is created in each service folder. 

```
#!/bin/bash -e
# Must be run in the service's directory.

help_message() {
  echo -e "Usage: $0 [apply|destroy|plan|refresh|show]\n"
  echo -e "The following arguments are supported:"
  echo -e "\tapply   \t Refresh the Terraform remote state, perform a \"terraform get -update\", and issue a \"terraform apply\""
  echo -e "\tdestroy \t Refresh the Terraform remote state and destroy the Terraform stack"
  echo -e "\tplan    \t Refresh the Terraform remote state, perform a \"terraform get -update\", and issues a \"terraform plan\""
  echo -e "\trefresh \t Refresh the Terraform remote state"
  echo -e "\tshow    \t Refresh and show the Terraform remote state"
  exit 1
}

apply() {
  plan
  echo -e "\n\n***** Running \"terraform apply\" *****"
  terraform apply
}

destroy() {
  plan
  echo -e "\n\n***** Running \"terraform destroy\" *****"
  terraform destroy
}

plan() {
  refresh
  terraform get -update
  echo -e  "\n\n***** Running \"terraform plan\" *****"
  terraform plan
}

refresh() {

  account=$(pwd | awk -F "/" '{print $(NF-4)}')
  region=$(pwd | awk -F "/" '{print $(NF-3)}')
  vpc=$(pwd | awk -F "/" '{print $(NF-2)}')
  environment=$(pwd | awk -F "/" '{print $(NF-1)}')
  service=$(pwd | awk -F "/" '{print $NF}')

  echo -e "\n\n***** Refreshing State *****"

  terraform remote config -backend=s3 \
                          -backend-config="bucket=${account}" \
                          -backend-config="key=${region}/${vpc}/${environment}/${service}/terraform.tfstate" \
                          -backend-config="region=us-east-1"
}

show() {
  refresh
  echo -e "\n\n***** Running \"terraform show\"  *****"
  terraform show
}

## Begin script ##
if [ "$#" -ne 1 ] || [ "$1" = "-h" ] || [ "$1" = "--help" ]; then
  help_message
fi

ACTION="$1"

case $ACTION in
  apply|destroy|plan|refresh|show)
    $ACTION
    ;;
  ****)
    echo "That is not a vaild choice."
    help_message
    ;;
esac
```

### Service Module Instantiation

In addition to remote.sh, the environment's service directory contains a [main.tf](https://github.com/TerraformDesignPattern/environments/blob/master/sysadvent-production/us-east-1/production-us-east-1-vpc/production/ssh-bastion/main.tf) file.

```
module "environment" {
  source = "../"
}

module "bastion" {
  source = "git@github.com:TerraformDesignPattern/bastionhost.git"

  aws_account      = "${module.environment.aws_account}"
  aws_region       = "${module.environment.aws_region}"
  environment_name = "${module.environment.environment_name}"
  hostname         = "${var.hostname}"
  image_id         = "${var.image_id}"
  vpc_name         = "${module.environment.vpc_name}"
}
```

We call two modules in `main.tf`. The first is an _environment module_, which we'll talk about in a moment, and the second is an _environment service module_. 

### Environment Service Module

_Environment service module?_ 

Everytime I've introduced this term, I've seen this... 

![alt tag](https://raw.githubusercontent.com/TerraformDesignPattern/SysAdvent2016/master/dogs.gif)

An environment service module, or _ESM_ for short, is just a way to specify, in conversation, that we are talking about the code that actually instantiates a service module.

If you look at the ESM declaration in `main.tf` above, you'll see it is using the output from the _enviroment module_ to define variables that will be passed into the _service module_. If we take a step back to review our directory structure we see the service we are deploying sits within the production environment's directory:

`sysadvent-production/us-east-1/production-us-east-1-vpc/production/ssh-bastion`

Within the production environment's directory is an [outputs.tf](https://github.com/TerraformDesignPattern/environments/blob/master/sysadvent-production/us-east-1/production-us-east-1-vpc/production/outputs.tf) file.

```
output "aws_account" {
  value = "production"
}

output "aws_region" {
  value = "us-east-1"
}

output "environment_name" {
  value = "production"
}

output "vpc_name" {
  value = "production-us-east-1-vpc"
}
```

We are able to create an entire service, regardless of resources, with a very generic ESM and just four values from our environment module. We are using our organization's defined and somewhat colloquial terms to create our infrastructure. We don't need to remember ARNs, ID's or other allusive information. We don't need to remember naming conventions either as the service module will take care of this for us.

### Service Module Example 

So far we've established a repeatable ways to run our Terraform command and guarantee that our state is managed properly and consistently. We've also instantiated a _service module_ from within an _environment service module_. We are now going to dive into the components of a service module.

A service module will be reused throughout your infrastrcuture so it must be generic and parameterized. The module will create all Terraform provider resources required by the service.

In my experience, I've found splitting each resource type into its own file improves readability. Below is the list of Terraform files from our [example bastion host service module repository](https://github.com/TerraformDesignPattern/bastionhost):

```
bastionhost
|-- data.tf
|-- ec2.tf
|-- LICENSE
|-- outputs.tf
|-- providers.tf
|-- route53.tf
|-- security_groups.tf
`-- variables.tf
```

The contents of most of these files will look pretty generic to the average Terraform user. The power of this pattern lies within the [data.tf](https://github.com/TerraformDesignPattern/bastionhost/blob/master/data.tf) as it allows the simplistic instantiation.

```
// Account Remote State
data "terraform_remote_state" "account" {
  backend = "s3"

  config {
    bucket = "${var.aws_account}"
    key    = "terraform.tfstate"
    region = "us-east-1"
  }
}

// VPC Remote State
data "terraform_remote_state" "vpc" {
  backend = "s3"

  config {
    bucket = "${var.aws_account}"
    key    = "${var.aws_region}/${var.vpc_name}/terraform.tfstate"
    region = "us-east-1"
  }
}
```

Sooooo. What populates the state file for the VPC data resources? Enter the  _VPC Service Module_ 

>There is no cattle.  
There are no layers.  
There is no spoon.  

Everything is just a compartmentalized service. The module that creates your VPC is a separate "service" that lives in its own repository.

We create our account resources (DNS zones, SSL Certs, Users) within an account service module, our vpc resources from a VPC module within a vpc service module and our services (Application Services, RDS, Web Services) within an Environment Service Module.

We use a [Bash wrapper](https://github.com/TerraformDesignPattern/environments/tree/master/templates) to publish the state of resources in a consistent fashion.

Lastly, we abstract the complexity of infrastructure configuration management by querying Terraform state files based on a strict S3 key structure.

## Real World Example

Follow along with the example by pulling the [TerraformDesignPattern/environments repository](https://github.com/TerraformDesignPattern/environments). The configuration of each module within the environment repository will consist of roughly the same steps: 

1. Create the required files. Usually `main.tf` and `variables.tf` or simply an `outputs.tf` file.
2. Populate `variables.tf`/`outputs.tf` with your desired values.
3. Create a symlink to a specific `remote.sh` (`account-remote.sh`, `service-remote.sh` or `vpc-remote.sh`) from within the appropriate directory.
  * For example, to create the `remote.sh` wrapper for your account service module, issue the following from within your `environments/$ACCOUNT` directory: `ln -s ../templates/account-remote.sh remote.sh`
4. Run `./remote.sh apply`

### Prerequisites

* __Domain Name__: I went over to [Namecheap.com](https://www.namecheap.com/) and grabbed the `sysadvent.host` domain name for $.88.  
* __State File S3 Bucket__: Create the S3 bucket to store your state files in. This should be the name of the [account folder](https://github.com/TerraformDesignPattern/environments/tree/master/sysadvent-production) within your [environments](https://github.com/TerraformDesignPattern/environments) repository. For this example I created the `sysadvent-production` S3 bucket.  
* __SSH Public Key__: As of the writing of this post, the `aws_key_pair` Terraform resource does not currently support creating a public key, only importing one.
* __SSL ARN__: [AWS Certificate Manager offers free SSL/TLS certificates](http://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request.html).

### Getting Started

This _Real World Example_ assumes you are provisioning a new AWS account and domain. For those working in a brown field, the following section provides a quick example of how to build a scaffolding that can be used to deploy the design pattern.

#### State Scaffolding or Fake It 'Til You Make It (With Terraform)

Within the [environments/sysadvent-production account directory](https://github.com/TerraformDesignPattern/environments/tree/master/sysadvent-production) is an [s3.tf file](https://github.com/TerraformDesignPattern/environments/blob/master/sysadvent-production/s3.tf) that creates a `dummy_object` in S3:

```
resource "aws_s3_bucket_object" "object" {
  bucket = "${var.aws_account}"
  key    = "dummy_object"
  source = "outputs.tf"
  etag   = "${md5(file("outputs.tf"))}"
}
```

A new object will be uploaded when `outputs.tf` is changed. This change updates the remote state file and thus any outputs that have been added to `outputs.tf` will be added to the remote state file as well. To use a resource (IAM Role, VPC ID, or Zone ID) that was not created with Terraform, simply add the desired data to the account, vpc, or ESM's `outputs.tf` file. Since not all resources can be imported via data resources, this enables us to migrate in small, iterable phases. 

In the example below, the AWS Account ID will be added to the account's state file via this mechanism. The `outputs.tf` file defines the account ID via the `aws_account` variable:

```
output "aws_account" {
  value = "${var.aws_account}"
}
```

### Stage One: Create The Account Service Module (ASM)

```
Working Directory: environments/sysadvent-production
```

As per the name, this is were account wide resources are created such as DNS Zones or Cloudtrail Logs. The `sysadvent-production` ASM will create the following:

* Cloudwatch Log Stream for the account's Cloudtrail
* Import a public key 
* Route53 Zone
* The "scaffolding" S3 _dummy_object_ from `outputs.tf` to publish:
  * AWS Account ID
  * Domain Name
  * SSL ARN
  
#### Populate Variables

Populate the variables in the account's [variables.tf](https://github.com/TerraformDesignPattern/environments/blob/master/sysadvent-production/variables.tf) file:

```
aws_account - name of the account level folder
aws_account_id - your AWS provided account ID
domain_name - your choosen domain name
key_pair_name - what you want to name the key you are going to import
ssl_arn - ARN of the SSL certificate you created, free, with Amazon's Certificate Manager
public_key - the actual public key you want to import as your key pair
```

#### Execute!

Once you have created a state file S3 bucket and populated the `variables.tf` file with your desired values run `./remote.sh apply`.

### Stage Two: Create The VPC Service Module (VSM)

```
Working Directory: environments/sysadvent-production/us-east-1/production-us-east-1-vpc
```

The [TerraformDesignPattern/vpc](https://github.com/TerraformDesignPattern/vpc) module creates the following resources:

* VPC
* Enable VPC Flow Logs
* Cloudwatch Log Stream for the VPC's Flow Logs
* Flow Log IAM Policies
* An Internet Gateway
* Three private subnets 
* Three public subnets with nat gateways and elastic IP addresses
* Routes, route tables, and associations 

#### Populate Variables

Populate the following variables in the VSM's [variables.tf](https://github.com/TerraformDesignPattern/environments/blob/master/sysadvent-production/us-east-1/production-us-east-1-vpc/variables.tf) file:

```
availability_zones
aws_region
private_subnets
public_subnets
vpc_cidr
vpc_name
```

#### Execute!

Once you have populated the `variables.tf` file with your desired values, create the resources by running `./remote.sh apply`.

### Stage Three: Create The Environment Module

```
Working Directory: environments/sysadvent-production/us-east-1/production-us-east-1-vpc/production
```

The _environment module_ stores the minimal amount of information required to pass to an _environment service module_. As mentioned previously, this module consists of a single [outputs.tf](https://github.com/TerraformDesignPattern/environments/blob/master/sysadvent-production/us-east-1/production-us-east-1-vpc/production/outputs.tf) file which requires you to configure the following: 

```
aws_account
aws_region
environment_name
vpc_name
```

### Stage Four: Create An Environment Service Module (ESM)

```
Working Directory: environments/sysadvent-production/us-east-1/production-us-east-1-vpc/production/elk
```

Congratulations, you've made it to the point where this wall of text really pays off. 

Create the [main.tf](https://github.com/TerraformDesignPattern/environments/blob/master/sysadvent-production/us-east-1/production-us-east-1-vpc/production/elk/main.tf) file. The ELK's ESM calls an _environment module_, an _ami_image_id module_ , and the _ELK service module_. The _environment module_ supplies environment specific data such as the AWS account, region, environment name and VPC name. This module data is, in turn, passed to the _ami_image_id module_. The _ami_image_id module_ will return AMI ID's based on the enviroment's region. 

The ELK ESM will create the following resources:

* Three instance ELK stack within the previously created private subnet.
* The Elasticsearch instances will be clustered via the EC2 discovery plugin.
* A public facing ELB to access Kibana.
* A Route53 DNS entry pointing to the ELB.

#### Execute!

Once you created the `main.tf` file, create the resources by running `./remote.sh apply`.

## Appendix

### Links

* [Terraforn Design Pattern Github Organization](https://github.com/TerraformDesignPattern)
  * [Bastion Host Service Module](https://github.com/TerraformDesignPattern/bastionhost)
  * [Cloud Trail Module](https://github.com/TerraformDesignPattern/cloudtrail)
  * [Environments Module](https://github.com/TerraformDesignPattern/environments)
  * [ELK Service Module](https://github.com/TerraformDesignPattern/elk)
  * [Packer Service Module](https://github.com/TerraformDesignPattern/packer)  
  * [VPC Module](https://github.com/TerraformDesignPattern/vpc)
* [Terraform Style Guide](https://github.com/jonbrouse/terraform-style-guide/blob/master/README.md)

### Gotchas

During the development of this pattern I stumbled across a couple _gotchas_. I wanted to share these with you but didn't think they were necessarily pertinent to an introductory article.

#### Global Service Resources

##### Example: IAM Roles

We want to keep the creation of IAM roles in a compartmentalized module but IAM roles are  global thus you can only create them once. Using _count_ and a lookup based on our region, we can tell Terraform to only create the IAM role in a single region.

Example `iam.tf` file:

```
...
count = "${lookup(var.create_iam_role, var.aws_region, 0)}"
...
```

Example `variables.tf` file:

```
...
variable "create_iam_role" {
  default = {
    "us-east-1" = 1
  }
}
...
```

#### Referencing an ASM or VSM From an ESM

The ASM and VSM create account and VPC resources, respectively. If you were to reference an ASM or VSM a la an environment module within an ESM, you'll essentially be attempting to recreate the resources originally created by the ASM and VSM. 
