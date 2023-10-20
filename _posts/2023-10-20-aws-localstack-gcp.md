---
layout: post
title:  "Exploring AWS using GCP and localstack"
date:   2023-10-20
categories: jekyll update
---

Audience: 
* You want to start using AWS Cloud.
* You are familiar with Google Cloud.
* You are familiar with gcloud, aws-cli, terraform, git,  docker, python, gh, GitHub.
* You want to start coding *without any costs* for now.
* You *do not want to use a credit card* to practice AWS.
* You are happy with aws-cli vs AWS management console

In this guide, we will 
* introduce AWS
* introduce localstack, an AWS service emulator on docker
* run a localstack instance using Google CLoud Shell
* configure localstack to run on GCP

## Introduction
In this guide, you'll leverage Google Cloud Shell as an environment to test a docker application that emulates AWS 
Cloud APIs. 

**Why Google Cloud Shell?**  

The main goal is to learn AWS Cloud (API calls/aws-cli/terraform provisioning) without a credit card:
* [Google Cloud Shell](https://console.cloud.google.com/home/) can be accessed with just a Google Account, no credit 
card, no upfront costs.
* Google Cloud Shell includes docker, terraform and can be set up in minutes
* The docker application that emulates AWS: [localstack](https://github.com/localstack/localstack) can be easily
installed using Cloud Shell as a development environment.

**Result**  
* You have an environment to practice AWS in minutes
* Especially useful to practice AWS resources provisioning with aws-cli and terraform, and IAM management.
* Main drawback: you do not have the AWS web management console

 
## Introducing AWS
### AWS Cloud overview
AWS Cloud is the Amazon Web Services public cloud offering. It is an alternative cloud solution to Google Cloud and a 
market rival.    
* AWS Cloud uses a substantially different model to Google Cloud for managing identities and resources.  

**Ways to interact with AWS services**

* [AWS Management Console](https://console.aws.amazon.com): A  web-based, graphical user interface to manage your 
AWS resources
* [Command-line interface](https://aws.amazon.com/cli/): **aws-cli**. AWS counterpart to Google Cloud **gcloud**
* [AWS Software Development Kits (SDKs)](https://aws.amazon.com/developer/tools/) allow developers to interact with AWS 
Cloud from programming code

**AWS account**  
An AWS account is the organizing entity for  resources, settings, permissions, and other metadata that describe your 
cloud applications, with a similar use case as a Google Cloud project(or set of projects with the same project owner).
* Resources within a single AWS account can work together easily, for example by communicating through an internal 
network, subject to certain rules.
* An  AWS account t can't access another AWS account resources by default. AWS IAM policies apply.  
*Note: for those familiar with Google Cloud IAM, AWS IAM and identity management is substantially different*

* To start using AWS Cloud you need an AWS account, which up to this date, although it provides an extensive Free tier,
**requires a credit card**. 

It is undeniable that at some point in their career, a developer, devops or cloud platform engineer must master several
cloud environments, one of them being AWS Cloud. 
So, what can you do if you need to:  
* learn AWS and *no credit card  available*,  
* or need to *learn services not on the Free tier* without spending, 
* or just incorporate AWS emulation or mock AWS services to CI/CD testing 


## Enters localstack...
Basically [localstack](https://github.com/localstack/localstack) is docker application that emulates AWS and 
can be installed on any environment that supports Docker, such as Google Cloud Shell.  

### Quick features summary
AWS service API emulator on docker
* Allows local AWS dev environments
* Supports aws-cli and AWS API calls overriding AWS public endpoints with local AWS endpoints running on docker.
* Works fine with AWS terraform provider.
* No AWS management console, though.
* Check [localstack list of supported features and AWS APIs](https://docs.localstack.cloud/user-guide/aws/feature-coverage/)
* Ideal for AWS learners or proof of concept projects.

To learn more, check the  [Getting Started guide](https://docs.localstack.cloud/getting-started).  

### Big thanks!  

Thanks to the people behind localstack: [Contributors](https://github.com/localstack/localstack/blob/master/README.md#contributors),
[Backers](https://github.com/localstack/localstack/blob/master/README.md#backers) and [Sponsors](https://github.com/localstack/localstack/blob/master/README.md#sponsors).

## Running localstack on GCP
### Create Google Cloud resources
1. Create a [Google Cloud](https://console.cloud.google.com/home/dashboard)  platform account if you do not already have it.
2. [Create a Google Cloud project](https://developers.google.com/workspace/guides/create-project) or use an existing one.

### Use Google Cloud Shell
[Google Cloud Shell](https://console.cloud.google.com/home/) is a VM and interactive shell environment for Google Cloud, 
accessible from the Google Cloud Console.
* Access to a VM and [public web accessible shell](https://shell.cloud.google.com/) 
* An interactive [Code Editor](https://ide.cloud.google.com)
* Other general development tools: docker, minikube, terraform, helm, etc.  
* Use [Cloud Shell Web Preview](https://cloud.google.com/shell/docs/using-web-preview) to publish applications running 
in Cloud Shell to a public URL (access limited to your Google Account).
* It also includes the [Google Cloud SDK](https://cloud.google.com/sdk/docs/quickstart)
* To start coding right away, launch [Google Cloud Shell](https://console.cloud.google.com/home/)

### Alternatively use a local dev environment
If not convinced to use Google Cloud Shell,  you can use a *local environment with docker, terraform,and python3*.  
* You can still follow the below instructions to configure localstack and terraform AWS provider.
* If in trouble, check the [localstack general installation documentation](https://github.com/localstack/localstack/blob/master/README.md#installation) as reference

### Install and configure aws-cli
```shell
# AWS-CLI installation  
export INSTALL_ZIP_URL="https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip"
cd ~
curl ${INSTALL_ZIP_URL} -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
# Output
# aws-cli/2.13.17 Python/3.11.5 Linux/5.15.120+ exe/x86_64.ubuntu.18 prompt/off

# Configure AWS-CLI
# Values extracted from localstack guide
aws configure
# Use values
# AWS_ACCESS_KEY_ID=[test]
# AWS_SECRET_ACCESS_KEY=[test]
# AWS_DEFAULT_REGION [us-east-1]
# Default Ouput Format [None]

# Install aws auto-complete
which aws_completer
# /usr/local/bin/aws_completer
complete -C '/usr/local/bin/aws_completer' aws
```

### Install localstack
```shell
# Install localstack using python local environment
# Create python local environment
cd ~
mkdir localstack-venv
cd localstack-venv
export VENV_PATH=$(pwd)
cd ~
python3 -m venv ${VENV_PATH}
source ${VENV_PATH}/bin/activate

# Install localstack
python3 -m pip install localstack


# Launch localstack
# /usr/local/bin/localstack start -d
localstack start -d
# Output
# 💻 LocalStack CLI 2.2.0
#
#[10:18:04] starting LocalStack in Docker mode 🐳                               localstack.py:409
#           preparing environment                                                bootstrap.py:623
#           configuring container                                                bootstrap.py:631
#           starting container                                                   bootstrap.py:638
#[10:18:40] detaching     
```
### How to run aws-cli commands
```shell
# Check the running localstack docker
# Notice the port redirection for AWS APIs 0.0.0.0:4566->4566/tcp
docker ps -l
# Output
# CONTAINER ID   IMAGE                   COMMAND                  CREATED         STATUS                   
# 981793108df1   localstack/localstack   "docker-entrypoint.sh"   3 minutes ago   Up 3 minutes (healthy)   
# PORTS                                                                NAMES 
# 0.0.0.0:4510-4559->4510-4559/tcp, 0.0.0.0:4566->4566/tcp, 5678/tcp   localstack_main


# Command example using localstack as AWS endpoint
# Use endpoint URL to change default AWS APIs listening addresses
export IAM_USER=test
aws iam create-user --user-name ${IAM_USER} --endpoint-url https://0.0.0.0:4566 --no-verify-ssl
# Output
# urllib3/connectionpool.py:1061: InsecureRequestWarning: Unverified HTTPS request is being made to host '0.0.0.0'. 
# Adding certificate verification is strongly advised. See: https://urllib3.readthedocs.io/en/1.26.x/advanced-usage.html#ssl-warnings
#
# {
#    "User": {
#        "Path": "/",
#        "UserName": "test",
#        "UserId": "w0vcxawa6jng1wn2vmbg",
#        "Arn": "arn:aws:iam::000000000000:user/test",
#        "CreateDate": "2023-10-20T10:39:25.060000+00:00"
#    }
# }
```

### Optional: install awscli-local
Use awscli-local so you do not need to add the endpoint flag everytime to the aws command

```shell
# awslocal is equivalent to running 
#       aws --endpoint-url https://0.0.0.0:4566 --no-verify-ssl
pip install awscli-local

export IAM_USER=test-2
awslocal iam create-user --user-name ${IAM_USER}
# Output
# {
#    "User": {
#        "Path": "/",
#        "UserName": "test-2",
#        "UserId": "tyc18o1rx0lxekhv2kpc",
#        "Arn": "arn:aws:iam::000000000000:user/test-2",
#        "CreateDate": "2023-10-20T10:41:51.197000+00:00"
#    }
# }
```

### How to use terraform to provision AWS resources on AWS stack
To use terraform with localstack, *configure the AWS terraform provider to use a different endpoint for AWS APIs*. Check the provider
 directive **endpoints** in provider.tf in the example below. 

```console
{
    networkmanager = "http://localhost:4566"
    ec2 = "http://localhost:4566"
  }
```
If you are using other type of AWS resources, add the name of the API, like 'ec2' to 'iam', 'lambda', etc. and the API 
endpoint URL, according to the type of resource.
Check [localstack list of supported features and AWS APIs](https://docs.localstack.cloud/user-guide/aws/feature-coverage/) 
and [AWS terraform provider custom endpoints](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/custom-service-endpoints)
```console
{
    networkmanager = "http://localhost:4566"
    ec2 = "http://localhost:4566"
    lambda = "http://localhost:4566"
  }
```

```shell
# Launch Google Cloud Shell
# Create a dir for your AWS TF assets
cd ~
mkdir aws-tf
cd aws-tf

# Create local repo for your terraform code
git init
nano provider.tf
nano ec2.tf
```

```console
# provider.tf
provider "aws" {
  region                      = "us-east-1"
  access_key                  = "test"
  secret_key                  = "test"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true
  endpoints {
    networkmanager = "http://localhost:4566"
    ec2 = "http://localhost:4566"
  }
}

# ec2.tf
resource "aws_vpc" "my_vpc" {
  cidr_block = "172.16.0.0/16"

  tags = {
    Name = "tf-example"
  }
}

resource "aws_subnet" "my_subnet" {
  vpc_id            = aws_vpc.my_vpc.id
  cidr_block        = "172.16.10.0/24"
  availability_zone = "us-west-2a"

  tags = {
    Name = "tf-example"
  }
}

resource "aws_network_interface" "foo" {
  subnet_id   = aws_subnet.my_subnet.id
  private_ips = ["172.16.10.100"]

  tags = {
    Name = "primary_network_interface"
  }
}

resource "aws_instance" "foo" {
  ami           = "ami-005e54dee72cc1d00" # us-west-2
  instance_type = "t2.micro"

  network_interface {
    network_interface_id = aws_network_interface.foo.id
    device_index         = 0
  }

  credit_specification {
    cpu_credits = "unlimited"
  }
}
```

```shell
gh auth login
export GH_REPO=my-aws-tf
gh repo create ${GH_REPO} --private

# Add newly created repository to remotes list
export GIT_REMOTE_URL="https://github.com/${GITHUB_USER}/${GH_REPO}.git"
export GIT_REMOTE_ID=origin-GH-${GH_REPO}
git remote add ${GIT_REMOTE_ID} ${GIT_REMOTE_URL}
git remote -v

# And start committing and planning/applying as required.
git commit -a  
terraform init
terraform plan
# ...

# Output
  # ...
 # aws_instance.foo will be created
 # + resource "aws_instance" "foo" {
 #     + ami                                  = "ami-005e54dee72cc1d00"
 #     + arn                                  = (known after apply)
 #     + associate_public_ip_address          = (known after apply)
 #     ...
 #      + instance_type                        = "t2.micro"
 #      ...
 #      + network_interface {
 #          + delete_on_termination = false
 #          + device_index          = 0
 #          + network_card_index    = 0
 #          + network_interface_id  = (known after apply)
 #        }
 #   ...
```
**Reusing localstack terraform code**  
* Do not forget to remove or reconfigure the endpoints directive in terraform aws provider if you ever deploy your 
authored terraform code on the real AWS Cloud.
```console
  endpoints {
    networkmanager = "http://localhost:4566"
    ec2 = "http://localhost:4566"
  } 
```

### Wrapping up
At this point you can learn to develop and test AWS resources on a free, fast to set up environment, accessible over the
internet, no credit card needed.
And to top it up, you can even use the Google Cloud integrated IDE, [Code Editor](https://ide.cloud.google.com), which 
has a nice Terraform helper plugin.  

![AWS localstack terraform on GCP Theia](/blog/res/img/theia-ide-tf.jpg)

