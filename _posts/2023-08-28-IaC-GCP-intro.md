---
layout: post
title:  "Introduction to IaC with Terraform in GCP"
date:   2023-08-28
categories: jekyll update
---

Audience:
* You want to apply Infrastructure as Code to your Google Cloud projects
* You are familiar with gcloud, terraform, git
* You want to start coding *without any costs* for now.

In this guide, we will 
* introduce the concept of Infrastructure as Code, IaC
* explore Terraform for Google Cloud resources
* use terraform and Google Cloud Platform to apply IaC procedures
* inspect basic GCP terraform examples


## IaC in Google Cloud Platform  

**IaC**  

  Infrastructure as code (IaC) allows quick and automated  provisioning, configuration, deployment and removing of 
infrastructure assets by using software tools capable of interpreting code that represents infrastructure resources and 
their configuration settings and use this code to generate requests to  infrastructure provisioners and managers, such 
as Google Cloud.
	The code representing the infrastructure and the provisioning operations can be stored in versioned repositories and
allows:  
* Building infrastructure resources on demand.
* Decommissioning  the infrastructure when not in use.
* Changing configuration settings when required in an agile way.
* Creating identical infrastructures for dev, test, and prod
  * Using templates to create different types of resources per group
* Integrating Infrastructure in a CI/CD pipeline.
* Using IaC templates in automated disaster recovery procedures.
* Strict infrastructure resources usage control, traceability  and accountability.

**GCP IaC tools**  

Several tools can be used for IaC in GCP. Google Cloud supports, among others:  
* [Deployment Manager](https://cloud.google.com/deployment-manager/docs)
* [Terraform](https://cloud.google.com/docs/terraform) 

Deployment Manager relies on YAML, Python, and Jinja 2 templates to describe resources, whilst Terraform has its own 
JSON based syntax. 

In both of these IaC tools, infrastructure resource deployments are described in a set of files known as a 
**configurations**, specifying the resources that should be provisioned. Configurations can be modularized using 
**templates** in Deployment Manager or **modules** in Terraform, allowing abstraction of resources into reusable 
components across deployments.  
 
**Terraform**  

Terraform is a tool that uses as input a number of files describing infrastructure resources, called **configurations**,
to generate requests to the resources providers. The basic request is a  request to provision those resources.  

Once resources are provisioned, Terraform stores information about those resources and their settings and status. This 
information, representing the managed infrastructure is called Terraform state.  

Using the current state and the configurations, Terraform can be used to manage transitions in the state: adding more 
resources, changing settings and properties, decommissioning resources, etc. by running 
[Terraform commands](https://cloud.google.com/docs/terraform/basic-commands)   

Terraform is compatible with many [providers](https://registry.terraform.io/browse/providers), on-premises and Cloud: 
VMware, Kubernetes, Dockers, Google Cloud, Amazon, OpenStack, and many more.

## Basic IaC management procedures
### Apply source control to Infrastructure resource definitions
In the example, the configuration defines a simple virtual machine that will be managed with terraform. 
It is expected that new resources will be added to the project, and you want to start managing the project resources 
with IaC procedures.
 
**Store terraform configurations in a code repository**
```shell
# Create or clone a git repo
# Checkout a new branch to introduce code changes
export REPO=iac_basics
mkdir ${REPO}
cd ${REPO}
git init 
git checkout -b FEATURE_BRANCH

# Write your code
# Create or modify terraform configuration in source code
nano instance.tf

# instance.tf
  resource "google_compute_instance" "terraform_vm" {
    project      = "<PROJECT_ID>"
    name         = "tf-managed-vm-101"
    machine_type = "n1-standard-1"
    zone         = "us-west1-c"
    boot_disk {
      initialize_params {
        image = "debian-cloud/debian-11"
      }
    }
    network_interface {
      network = "default"
      access_config {
      }
    }
  }

# Add to git branch to track changes
git add instance.tf
```

**Test your code**  
  Testing IaC code involves creating infrastructure by calling provisioning systems and checking the resources have been
provisioned successfully.

 Ideally the code should be tested by a CI system in a testing environment. The environment is defined by the account 
and system variables set  when running the command 'terraform'.
  In an automated  CI/CD process, an agent would be launched with the appropriate environment context and automatically 
execute the tests and validate the results.  

The example shows how to manually deploy the infrastructure resources to the current environment with terraform and how 
to generate a plan file. Plan files describe the provisioning process and can be used as advanced test inputs and 
considered artifacts in the IaC CI/CD process.


```shell
# Test your IaC code
# Terraform setup
terraform init

# Plan resources deployment, optionally save plan
terraform plan [-out PLAN_FILE]
Terraform will perform the following actions:
  # google_compute_instance.terraform_vm will be created
  + resource "google_compute_instance" "terraform" {
      ...
      + name         = "tf-managed-vm-101"
      + instance_id  = (known after apply)
      ...
    }
Plan: 1 to add, 0 to change, 0 to destroy.

# Check state before deploying
terraform  show
# Ouput
  No state.

# Deploy resources generating a new plan or from saved plan
terraform apply ["plans/PLAN_FILE"]
# Output
  ...
  Plan: 1 to add, 0 to change, 0 to destroy.
  Do you want to perform these actions?
    Terraform will perform the actions described 
  ...

# Check resources have been provisioned
# Check that a VM has been created by terraform with specified property values
gcloud compute instances list --project PROJECT_ID

# Check state after deploying
terraform  show
  ...
  + resource "google_compute_instance.terraform_vm " {
        ...
        + name         = "tf-managed-vm-101"
        + instance_id  = "3408292216444307052"
        ...
      }
  ...

# Commit changes to git repository
git commit -m "Initial commit. Terraform VM in PROJECT_ID"
```

From this point,  it is possible to manage resources in terraform state by  modifying configurations with versioned 
source control and terraform plan/apply.
It is possible to maintain separate branches or repos for several environments. The goal is to apply CI/CD operations to 
the infrastructure defining code.

### Code Packaging and Reusability
Terraform allows code packaging and reusability by providing parameterizable 
[modules](https://developer.hashicorp.com/terraform/language/modules).  
In the example:
* a module previously defined by a 3rd party and called *gcs-static-website-bucket* 
* stored in folder /modules
* is reused by a Terraform configuration (web-bucket.tf)
* that provides inputs to the module via terraform variables. 
Effectively the current code contributor has only written the *web-bucket.tf* file that indicates which module to use, 
previously authored , and the *terraform.tfvars* file defining the values for the module inputs.

Apart from local stored modules, terraform allows reusing modules from several 
[remote sources](https://developer.hashicorp.com/terraform/language/modules/sources)

```console
# Modules
# Reusing 3rd party source code to create resources, passing arguments

# web-bucket.tf
module "gcs-static-website-bucket" {
  source = "./modules/gcs-static-website-bucket"
  input_n    = var.bucket_name
  input_p 	 = var.project_id
  input_l    = var.bucket_location
  input_t    = var.bucket_topic 
}

# terraform.tfvars
variable "bucket_name" {
  type = string
  default = "website"
}
...
variable "bucket_topic" {
  type = string
  default = "uploads"
}
```

Although  not relevant to our case in point, describing IaC procedures, just a brief description of the module function,
which is related to a well known use case of 
[Google Cloud Storage and PubSub](https://cloud.google.com/storage/docs/pubsub-notifications).  
The reusable module defines two resources designed to be associated one to each other and managed as a single unit:
* a Google Cloud Bucket storing a static website 
* with an associated PubSub topic, meant to receive asynchronous messages when the bucket content is updated.
 
```console
# Module code
# ./modules/gcs-static-website-bucket/main.tf
resource "google_storage_bucket" "bucket" {
  name               = var.input_n
  project_id         = var.input_p
  location           = var.input_l
  ...
}

resource "google_pubsub_topic" "web_update" {
  name = var.input_t
}
```

To apply the configuration *web-bucket.tf*, run terraform from the folder where the configuration is located. 


```shell
ls 
  web-bucket.tf
  terraform.tfvars

# terraform reads web-bucket.tf and re-uses the  code located in ./modules/gcs-static-website-bucket/main.tf
# to provision the resources defined in the configuration using as module inputs the variables defined in 
# variable file terraform.tfvars

terraform plan
...
  module.gcs-static-website-bucket.google_storage_bucket.bucket   will be created
  module.gcs-static-website-bucket.google_pubsub_topic.web_update will be created
...
```


### Incorporating existing resources to terraform state
Terraform allows provisioning, managing and decommissioning  infrastructure resources from configurations. But more 
often than not, the need exists to add resources already in place but not yet managed by terraform. The operation to 
incorporate existing resources to 
terraform is called [importing resources](https://developer.hashicorp.com/terraform/language/import).

The following commands show the basics of the import operation, using existing GCP resources:  


* List existing resources in a GCP project
```shell
gcloud pubsub topics list --project=PROJECT_ID
name: projects/PROJECT_ID/topics/orders
```

* Write configurations with minimal properties matching existing resources  

```shell
nano topics.tf
# topics.tf
resource "google_pubsub_topic" "tf_topic_1" {
  name = "orders"
}
```
* Import GCP resource to terraform state using Google Cloud API resource unique ID.  

To see minimal properties for a specific GCP resource, check the 
[Terraform Google Cloud Platform Provider ](https://registry.terraform.io/providers/hashicorp/google/latest/docs).

```shell
# Initially there are not any resources being managed by terraform
terraform init
terraform  show
No state.

# Syntax:
# terraform import TF_RESOURCE_TYPE.TF_RESOURCE_NAME  API_OBJECT_UNIQUE_ID
terraform import   google_pubsub_topic.tf_topic_1     projects/PROJECT_ID/topics/orders

terraform  show
# Note the resource is now part of tf state. 
# Properties have been populated from actual API object
resource "google_pubsub_topic" "tf_topic_1" {
    id      = "projects/PROJECT_ID/topics/orders"
    labels  = { "env" = "dev" }
    name    = "orders"
    project = "PROJECT_ID"
    timeouts {}
}
```
* Update the configuration files created in the initial step to include the new populated properties: importing the state does not change the resource configuration file, the resource configuration still needs to be declared.
   
* Mind that there might be property values present in the state that might be transient or derived from others and don't need to be 
included in the configuration files.  For instance, including both the name and project, or providing the id for a topic 
is redundant.   
* Use the [Terraform Google Cloud Platform Provider ](https://registry.terraform.io/providers/hashicorp/google/latest/docs) 
as reference when writing configurations.

```shell
  nano topics.tf
# topics.tf
resource "google_pubsub_topic" "tf_topic_1" {
  name = "orders"
  labels  = { "env" = "dev" }
...
}

# Commit to git repository current branch
git commit -a -m "Added existing topic to managed infrastructure:   projects/PROJECT_ID/topics/orders""
```
### Destroying infrastructure in current terraform state
The command *terraform destroy* decommissions or delete all the resources in the Terraform state. For all practical 
purposes, the resources cease to exist. 
To decommission a specific provisioned resource, it is possible to target just the one resource:  

```console
terraform destroy  --target TF_RESOURCE_TYPE.TF_RESOURCE_NAME
```

Below there's  an example of how to manage IAM bindings for a project with terraform to illustrate the 'destroy' 
command and targeting terraform resources.  
Suppose you are a Cloud Engineer who has been assigned managing project resources, including IAM, with Terraform. Your 
first task is to apply IAM best practices and 
[limit the use of basic IAM roles to access a project](https://cloud.google.com/iam/docs/using-iam-securely#least_privilege).

```shell
# Capturing existing role binding to manage it with terraform
export PROJECT_ID=<YOUR_PROJECT_ID>
export ROLE_ID='roles/viewer'

# Terraform uses the cloudresourcemanager API to manage IAM, so the first step is to enable API
# Reference: Terraform Google Cloud Platform Provider, https://registry.terraform.io/providers/hashicorp/google 
gcloud services enable cloudresourcemanager.googleapis.com

mkdir tf_iam
cd tf_iam

# Write config file with basic resource definition
nano main.tf
# main.tf
  resource "google_project_iam_binding" "viewers_binding" {
    # (resource arguments)
  }
  
terraform init

terraform import  google_project_iam_binding.viewers_binding "$PROJECT_ID $ROLE_ID"
# Output
    google_project_iam_binding.viewers_binding: Import prepared!
      Prepared google_project_iam_binding for import
      ...   
    Import successful!
    The resources that were imported are shown above. These resources are now in
    your Terraform state and will henceforth be managed by Terraform.
    
# Shows the current state
terraform show
# Output
    # google_project_iam_binding.viewers-binding:
    resource "google_project_iam_binding" "viewers_binding" {
        id      = "YOUR_PROJECT_ID/roles/viewer"
        members = [
            "user:someone@example.com",
        ]
        project = "YOUR_PROJECT_ID"
        role    = "roles/viewer"
    }


# Update the configuration file to populate properties read from state
nano main.tf
    resource "google_project_iam_binding" "viewers_binding" {
       members = [
            "user:someone@example.com",
        ]
        project = "YOUR_PROJECT_ID"
        role    = "roles/viewer"
    }

# Once imported, it can be managed with terraform by editing configuration and running terraform apply
# Ideally the terraform configurations should be added to source repositories and use change management procedures

# Adding a user can be achieved by editing the members property and then running terraform apply

# To remove the binding completely, targeting the specific terraform resource:
terraform destroy  --target google_project_iam_binding.viewers_binding
# Output
    Terraform will perform the following actions:
    google_project_iam_binding.viewers-binding will be destroyed
    ...


# Check IAM binding has been removed
export IDENTITY="someone@example.com"
export MEMBER="user:$IDENTITY"

gcloud projects get-iam-policy $PROJECT_ID --format "json(bindings.filter(members:"$MEMBER")).filter(role:roles\viewer))"
{
  "bindings": []
}
```


### Saving state to existing GCS bucket
By default, Terraform stores the current infrastructure state  in a local file named terraform.tfstate.  
You can define a [backend](https://developer.hashicorp.com/terraform/language/settings/backends/configuration#using-a-backend-block), 
which is a block of code in a configuration file defining Terraform metadata, to change the default state storage 
location.

A local backend is sufficient for a single operator, however, in a team, the state must be shared to provide a single 
source of truth for the Terraform managed infrastructure. Configuring the location of the state is a feature of 
Terraform backends, and there are several 
[available options](https://developer.hashicorp.com/terraform/language/settings/backends/gcs). 

When using Google Cloud, it is recommended to use a 
[Google Cloud Storage backend](https://developer.hashicorp.com/terraform/language/settings/backends/gcs) that, apart from providing shared storage for 
all team members, can provide 
[Identity and Access Management features](https://cloud.google.com/storage/docs/access-control/iam-roles), 
[versioning](https://cloud.google.com/storage/docs/object-versioning) and 
[multi-regional](https://cloud.google.com/storage/docs/locations) 
redundancy for the [Terraform state](https://developer.hashicorp.com/terraform/language/state)

**Storing Terraform state in Google Cloud Storage using a 'gcs' backend** 
```console
# main.tf
provider "google" {
  project     = var.project_id
  region      = var.region
}
resource "google_storage_bucket" "gcs-tf-state" {
  name        = var.bucket
  location    = var.bucket_location
  uniform_bucket_level_access = true
}
terraform {
  backend "gcs" {
    bucket  = var.bucket
    prefix  = "terraform/state"
  }
}
```

## Cloud Foundation toolkit
The [Cloud Foundation Toolkit](https://cloud.google.com/foundation-toolkit) provides a series of open source reference 
templates or blueprints for Terraform which reflect 
[Google Cloud best practices](https://cloud.google.com/docs/terraform/best-practices-for-terraform), stored in a [public 
GitHub repository](https://github.com/terraform-google-modules),  and can be used to quickly build a repeatable 
production ready Google Cloud setup.  

* To start working with it, fork the CFT to your own repository and modify as needed.
* The templates can be used independently and are easily customizable.   
* Using the  CFT provides consistency across teams and best practices out of the box.


