---
layout: post
title:  "CI: Automating builds with GitHub Actions"
date:   2023-09-04
categories: jekyll update
---

Audience: 
* You want to start using CI/CD procedures in Google Cloud Platform.
* You are familiar with CI procedures.
* You are familiar with gcloud, git,  docker, python, gh, GitHub.
* You want to start coding *without any costs* for now. 

In this guide, we will 
* revise the CI build process
* explore GitHub Actions
* explore GitHub events
* use GitHub Actions to automate a branch build and automated testing


## CI basics
Following  the concepts explored in the 
[CI](https://cloud.google.com/architecture/devops/devops-tech-continuous-integration) build example script used in posts:  
* [CI basics: creating build from a git branch](/blog/jekyll/update/2023/08/09/CI-intro-build-1)
* [CI basics: making tests an integral part of the build](/blog/jekyll/update/2023/08/09/CI-intro-build-2)
* [CI basics: Managing artifacts](/blog/jekyll/update/2023/08/09/CI-intro-build-3)

we will continue considering ways of triggering the build process everytime  there is a  change on a tracked git code 
branch.
 
## Bootstrap
The goal of the boostrap procedure is to create a repo structure
to simulate a CI system typical repo configuration and procedure:  
* A team repository in GitHub with a main branch
* One or more contributors local git repositories configured with the team repository as remote
* Every contributor tracking their own feature branches  and creating pull requests  to merge into the main branch

In the bootstrap procedure a remote Git repo in GitHub plays the part of the team repository.  
A local repository tracking the remote one is created to be used by a contributor to develop feature branches.

```shell
# START OF Bootstrap.........................................................................
# To populate both repos (team's and contributor's)  with an intial version of code
# an existing 3rd party repo example is cloned

# Get example source code
export SOURCE_REPO_NAME=py-holder
export SOURCE_REPO='https://github.com/amesones-dev/py-holder.git'
export SOURCE_BRANCH='main'

# Clone only SOURCE_BRANCH
git clone $SOURCE_REPO -b $SOURCE_BRANCH

# Create brand new repo with downloaded example

# Git current contributor details
export GIT_USER_NAME=<YOUR GIT USER NAME>
export GIT_USER_EMAIL=<YOUR EMAIL>

git config --global user.name ${GIT_USER_NAME}
git config --global user.email ${GIT_USER_EMAIL}
git config --global init.defaultBranch main

cd ${SOURCE_REPO_NAME}

# Initialize git 
git init

export ROOT_BRANCH='main'
git branch -m ${ROOT_BRANCH}

# Add files to branch
git add . && git commit -m "initial commit"

git status
# On branch main
# Your branch is up to date with 'origin/main' [EXAMPLE SOURCE REPO}
# nothing to commit, working tree clean

# Remove example source remote, only used to populate local repo with code, no longer needed
git remote remove origin
```

A GitHub  repo will be created to be used as the team repo, where all contributors will publish new features and merge into the main version of the code.

It is possible to use any other Source Code Repository product, check every individual product on how to create a team repo.  
* [GitHub](https://github.com/)
* [BitBucket](https://bitbucket.org/)
* [Google Cloud Repositories](https://cloud.google.com/source-repositories/docs)
* [AWS CodeCommit](https://aws.amazon.com/codecommit/)


```shell
# Create remote GitHub repository (to simulate team repo)
# Include note on how to login to Github
# Mind that GITHUB_USER might be different to GIT_USER_NAME
export GITHUB_USER=<YOUR GITHUB USER>

gh auth login
export TEAM_REPO=py-holder
gh repo create ${TEAM_REPO} --private
# Output
# ✓ Created repository <YOUR GITHUB_USER>/py-holder on GitHub


# Add newly created repository to remotes list
export GIT_REMOTE_URL="https://github.com/${GITHUB_USER}/${TEAM_REPO}.git"
export GIT_REMOTE_ID=github-team-repo
git remote add ${GIT_REMOTE_ID} ${GIT_REMOTE_URL}
git remote -v
# github-team-repo        https://github.com/<YOUR GITHUB_USER>/py-holder.git (fetch)
# github-team-repo        https://github.com/<YOUR GITHUB_USER>/py-holder.git (push)

# Sync code in the remote repo: push your main branch to team-repo
git push --set-upstream ${GIT_REMOTE_ID} main
# Output
# ...
#To https://github.com/<YOUR GITHUB_USER>/py-holder.git
# * [new branch]      main -> main


# END OF Bootstrap...........................................................................
```
At this point the needed elements are in place to start applying CI procedures:  
* A team repository in GitHub with a main branch
* A contributor local git repository configured with the same code as the remote team repository


## Source Repository events and webhooks
The key component that allows code build automation in CI systems is the ability of team Source Repository software to 
produce subscribable events when changes are made to branch code:  
* When a contributor runs an operation on a repository, the repository generates an event.
* CI systems can use requests to retrieve those events by using a well known API interface published by the team source 
repository software.
* CI systems use those events as triggers to run certain actions: building, testings, producing documentation based on 
changes, etc.

**Webhooks**
A webhook is a convenience feature built on top of team source code repo events APIs, that allows a remote system being 
notified on a certain URL(where an HTTP server is running ready to process notification requests) whenever an event from 
a predefined set occurs, instead of having to poll the events APIs manually.


**Example**  
* [GitHub Events](https://docs.github.com/en/rest/activity/events)
* [GitHub WebHooks](https://docs.github.com/en/webhooks/about-webhooks)
* [BitBucket WebHooks](https://support.atlassian.com/bitbucket-cloud/docs/manage-webhooks/)
* [Google Cloud Source Repositories](https://cloud.google.com/source-repositories/docs/code-change-notification)
* [Cloud Build WebHooks](https://cloud.google.com/build/docs/automate-builds-webhook-events?generation=2nd-gen)

For specific information, check documentation for compatibility with Team Source repositories for a specific CI system.  

**Popular CI systems**  
 
* [GitLab](https://about.gitlab.com/)
* [TeamCity](https://www.jetbrains.com/teamcity/)
* [CircleCI](https://www.jetbrains.com/teamcity/)
* [Travis CI](https://www.travis-ci.com/)
* [Google Cloud CI/CD](https://cloud.google.com/docs/ci-cd)
* [AWS CI/CD](https://docs.aws.amazon.com/whitepapers/latest/cicd_for_5g_networks_on_aws/cicd-on-aws.html)

## GitHub Actions
[GitHub Actions](https://docs.github.com/en/actions) (**GHA**) is a GitHub integrated product designed to run basic CI/CD 
procedures on GitHub source repositories.  

It's a great starting point to implement CI if your team code is stored in GitHub:
* Easily configurable: adding workflow files in yaml format to your source code
* Standard syntax, similar to any other commercial CI system
* Support for many source repositories events 
* No need to use webhooks or GitHub events API directly: integrated event management and subscription.
* Jobs management interface integrated with GitHub
* Provides runner agents to execute jobs using GitHub own compute resources
* Provides storage for your temporary artifacts, logs and logs history.
* Great documentation and extended users community
* You can start using it for free, check 
[Usage limits, billing, and administration](https://docs.github.com/en/actions/learn-github-actions/usage-limits-billing-and-administration) for details.
 

## Automated build with GitHub Actions

Following with the example, we are going to inspect the procedure followed by a contributor to implement automated 
testing and build of a new feature branch on a team repo with [GitHub Actions](https://docs.github.com/en/actions). 

### GitHub Actions basics  

* GitHub Actions workflow files are yaml files, stored in repo code under a certain path:
```console
.github/workflows/
  ...
  .github/workflows/build.yml
  .github/workflows/test.yml
  .github/workflows/tagging.yml
  ...
```
* The files include information about what event triggers the workflow and the steps to run when triggered

**Example**
```console
name: Run automated unit tests
on:                                 
  push:                             # Event (a contributor push to the repo)
    branches:                       # Branch filter (wich branches trigger the workflow)
      - '**'                        # In this case, every branch except main       
      - '!main'    
      
jobs:                               # What to do when triggered
  build:                            # Job name or id
    runs-on: ubuntu-latest          # Runner that runs the Job (agents are sets of Docker images and contexts)

    steps:
    - uses: actions/checkout@v3     # Use a predefined Action (checking out current branch)
                                    # Or name and define a run step
    - name: Build the Docker image   
      run: docker build . --file ./run/Dockerfile-tests   --tag  my-image:(date +%s)
```
Note these object names have a specific meaning in the context of GitHub Actions, and will be considered later in this 
guide:
* [Workflow](https://docs.github.com/en/actions/using-workflows/about-workflows)
* [Job](https://docs.github.com/en/actions/using-jobs/using-jobs-in-a-workflow)
* [Event](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)
* [Runner](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners)
* [Action](https://docs.github.com/en/actions/creating-actions/about-custom-actions)

### GitHub Actions triggering requirements

To use GitHub Actions:  
* a GitHub Actions workflow file must exist on the main branch.

To use GitHub Actions and specific branches triggers:  
* a GitHub Actions workflow file must exist on the main branch
* a GitHub Actions workflow file must exist on the branch that the trigger applies to

When an event is triggered for a branch, the .yml files used by GitHub Actions are the ones stored in said branch.
 
*Recommendation*

  * Use a basic GitHub action file on main branch to activate GitHub Actions and to use as template for branches when 
  checking out new feature branches 
  * Use additional customized .yml files on any other branches 
  

### Enabling GitHub Actions on team repo
Following the example  we will create a basic workflow called *event.yml* on the main branch to activate GitHub Actions 
for the team repo.
Every time a contributor feature branch is created, this file  can be used as template for customized branch actions.


The GitHub Actions job steps have been chosen to showcase the underlying elements of the automated triggering system:  
* The workflow prints the GitHub event that triggers it: 
  * the event is  generated by GitHub (Team Source Repository) 
  * if using a different CI system to GitHub Actions, the event would have been received 
    * by polling [GitHub events API](https://docs.github.com/en/rest/activity/events) 
    * or using [GitHub Webhooks](https://docs.github.com/en/webhooks/about-webhooks)
* It also prints the GitHub related environment variables: available when using GitHub Actions for CI procedures.
```console
# References 
# https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions

git status
# On branch main
# Your branch is up to date with 'origin/main' [EXAMPLE SOURCE REPO}
# nothing to commit, working tree clean

git remote -v
# github-team-repo        https://github.com/<YOUR GITHUB_USER>/py-holder.git (fetch)
# github-team-repo        https://github.com/<YOUR GITHUB_USER>/py-holder.git (push)

# On top repo level

mkdir .github
mkdir .github/workflows

# Create a GitHub Action file 
nano  .github/workflows/event.yml
```

```console
# .github/workflows/event.yml
name: Inspect GITHUB events and GitHub actions environment

on:
  push:
    branches:
      - '**'
      - '!main'
jobs:
  printenv:
    runs-on: ubuntu-latest
    steps:
    - name: GITHUB event
      run:  cat "$GITHUB_EVENT_PATH"
    - name: GITHUB env
      run:  printenv |grep GIT
```
**Notes on GitHub Action Workflow file**  

* The GHA Workflow triggers on [push](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#push) for
all branches except main, following
[branch trigger filtering rules](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#onpull_requestpull_request_targetbranchesbranches-ignore).  

* Prints the event (json file) that triggered the action and the generally available GitHub environment variables.

### Triggering GitHub Actions 
Let's examine how GitHub Actions would be triggerd during usual contributor routine, whenever creating a new feature branch
and pushing code to team repo.

**Create a new feature branch, change code and push to team repo**
```shell
# Step 1. Create a feature branch
export RID=$RANDOM
export FEATURE_BRANCH=new-feature-${RID}

git checkout -b $FEATURE_BRANCH
# Output
# Switched to a new branch 'new-feature-21455'

# The branch code includes the GitHub Actions seed file.

# Make a change or create a new file and push to remote
# to trigger GitHub Action.

# For example
#  Replace current default port in line in src/start.py
#     server_port = os.environ.get('PORT', '8080')

export SEARCH_STR="8080"
export REPLACE_STR="8081"
sed -i "s#${SEARCH_STR}#${REPLACE_STR}#g" src/start.py

git commit -m "Update default PORT in src/start.py"
git push origin $FEATURE_BRANCH
```
**Explore GHA Workflow runs history**  
* Navigate to your team repo GitHub project, created in the Bootstrap procedure.
```console
export GIT_REMOTE_URL="https://github.com/${GITHUB_USER}/${TEAM_REPO}.git"
```  

* Find the *.github/workflows/event.yml* file
* Open it and observe the *View Runs* icon in the top left corner  

![GitHub Action](/blog/res/img/gha-event.png)
  
* Click on the "View Runs" and then on the "Update default PORT in src/start.py" commit
* Observer the job status ("Success") the option to re-run it and the job label in green.  
* Click on the job label to inspect the job steps
 
![Job Result](/blog/res/img/gha-event-2.png)
 
* Click on each step and inspect the event and environment.

## Creating a GHA Workflow to run tests on feature branch
**Workflow Environment variables**
* Scope: accessible from every job in workflow
* Can be accessed as ${{ env.NAME }} in jobs
* Values are literal strings. When env.NAME is found in job definition
* ${{ env.NAME }} is replaced by strings specified in this section

```shell
# Step 1. Create a feature branch
git commit -m "Added GitHub action: run tests"
git push origin $FEATURE_BRANCH
```

```console
name: Run automated unit tests new-feature

on:
  push:
    branches:
      - '**'
      - '!main'

jobs:
  unittests:
    runs-on: ubuntu-latest
    env:
      TEST_ERR_CONDITION: FAILED
      TEST_OK: OK
      TEST_NOT_OK: NOT_OK

      TEST_DOCKER_TAG: test-${GITHUB_REPOSITORY_ID}-${GITHUB_REF_NAME}-$GITHUB_RUN_ID
      TEST_LOG: test-${GITHUB_RUN_ID}-result.log
      TEST_DOCKERFILE: ./run/Dockerfile-tests

    outputs:
      test-result: ${{ steps.test-report.outputs.result }}

    steps:
    - uses: actions/checkout@v3

    - name: Build the tests Docker image
      run: docker build . --file  ${{ env.TEST_DOCKERFILE }}  --tag ${{ env.TEST_DOCKER_TAG }}

    - name: Run unit tests
      run: docker run  ${{ env.TEST_DOCKER_TAG }} 2>&1 | tee ${{ env.TEST_LOG }}

    - id: test-report
      name: Generate test result outputs
      run: |
            found_errors=$(grep -o ${{ env.TEST_ERR_CONDITION }} ${{ env.TEST_LOG }} | head -n 1)
            if [ -z $found_errors ]; then result=${{ env.TEST_OK }};else result=${{ env.TEST_NOT_OK }};fi
            echo "result=${result}"
            echo "result=${result}" >> $GITHUB_OUTPUT

```

**Notes on GHA Workflow**  

* The action triggers on [push](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#push) for
all branches except main, following
[branch trigger filtering rules](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#onpull_requestpull_request_targetbranchesbranches-ignore).  

* Runs 1 jobs: **unittets** 
  * **unittests** uses a predefined Action (actions/checkout@v3) to check out the current branch code
  * **unittests** builds and run a docker image to run python unit tests as per branch code
  * **unittests** generates and output to be used by other jobs or workflows with the tests results


**About user environment variables**  

GHA provides different kinds of [variables](https://docs.github.com/en/actions/learn-github-actions/variables):
* Predefined environment variables
* User defined environment variables to be used by one or more workflows.

In the example, we will use user environment variables for a single workflow. 
Within a workflow, environment variables:  
* can have different 
[scopes](https://docs.github.com/en/actions/learn-github-actions/variables#defining-environment-variables-for-a-single-workflow): workflow, job, step.
* can be accessed as ${{ env.NAME }} within their scope

When ${{ env.NAME }}  is found in a workflow  file definition, is literally replaced by the literal string in their 
definition.
For example:
  TEST_LOG: test-${GITHUB_RUN_ID}-result.log

The .yml workflow is processed by GHA to generate scripts to be executed on the runners (shell scripts for a linux runner).  

Whenever the string *${{ TEST_LOG }}* is found in the yml, it is replaced by the literal  
*test-${GITHUB_RUN_ID}-result.log* in the resulting .sh script.  

During runtime a shell uses the replaced expression, which in this case is a dynamic expression whose effective value is
only known  at runtime



**About Predefined actions**  

**About Outputs** 
When jobs need to communicate between them, it is possible to define 
[jobs outputs](https://docs.github.com/en/actions/using-jobs/defining-outputs-for-jobs) by writing to a predefined 
stream referenced by the GHA environment variable *GITHUB_OUTPUT*  

```console
    job_generating_ouput  # Definition of job generating  ouput
                          # Output declaration, links name of output (test-result) 
                          # to a shell variable name and value written to $GITHUB_OUTPUT (result)
      outputs:                          
        test-result: ${{ steps.test-report.outputs.result }}
        
      steps:              # Job steps
        ...
          test-report:    # Step label used in output definition              
          ...
              # Shell variable link to output test-result in output declaration
              echo "result=OK" >> $GITHUB_OUTPUT
          ...
        ...
```

A job that needs another job output:

```console
    job_using_output                # Definition of job using another job ouput
    needs: job_generating_ouput     # Name of the job that generated the ouput
                                    # References the output using the output name
    ...
          echo "needs.job_generating_ouput.outputs.test-result"
    ...
    
```

  







**Notes on GHA Workflow**  

* The action triggers on [push](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#push) for
all branches except main, following
[branch trigger filtering rules](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#onpull_requestpull_request_targetbranchesbranches-ignore).  

* Runs 2 jobs: **unittets** and **branch_build**
  * **branch_build** depends on **unittets**
  * **branch_build** uses a **unittets** output, called test-result

* **unittests** builds and rund a docker image to run python unit tests as per branch code
 
* **branch_build** builds the application docker image if the tests finish wihout errors   





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
[remote sources](https://developer.hashicorp.com/terraform/language/modules/sources).

```console
# Modules
# Reusing 3rd party source code to create resources, passing arguments

# web-bucket.tf
module "gcs-static-website-bucket" {
  source = "./modules/gcs-static-website-bucket"
  input_n = var.bucket_name
  input_p = var.project_id
  input_l = var.bucket_location
  input_t = var.bucket_topic 
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
often than not, the need exists to add resources already in place but not yet managed by Terraform. The operation to 
incorporate existing resources to 
Terraform is called [importing resources](https://developer.hashicorp.com/terraform/language/import).

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

```shell
terraform destroy  --target TF_RESOURCE_TYPE.TF_RESOURCE_NAME
```

Below there's  an example of how to manage IAM bindings for a project with terraform to illustrate the *destroy* 
command and targeting terraform resources.  
Suppose you are a Cloud Engineer who has been assigned managing project resources, including IAM, with Terraform. Your 
first task is to apply IAM best practices and 
[limit the use of basic IAM roles to access a project](https://cloud.google.com/iam/docs/using-iam-securely#least_privilege).

```shell
# Capturing existing role binding to manage it with terraform
export PROJECT_ID=<YOUR_PROJECT_ID>
export ROLE_ID='roles/viewer'

# Terraform uses the cloudresourcemanager API to manage IAM,
# so the first step is to enable API
# Reference: Terraform Google Cloud Platform Provider
# https://registry.terraform.io/providers/hashicorp/google 
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
# iam_bindings ID syntax reference
# Reference: Terraform Google Cloud Platform Provider
# https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/google_project_iam.html
  
export API_OBJECT_UNIQUE_ID="${PROJECT_ID} ${ROLE_ID}"

terraform import google_project_iam_binding.viewers_binding ${API_OBJECT_UNIQUE_ID}
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


