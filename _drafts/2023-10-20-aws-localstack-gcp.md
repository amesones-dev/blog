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

# Create a GitHub Action workflow file. See content below
nano  .github/workflows/event.yml

# Commit and push to main in remote
git commit -m "Added GitHub action workflow file: /event.yml"
git push ${GIT_REMOTE_ID}
```

**.github/workflows/event.yml**  

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

```shell
# Step 1. Create a feature branch
# Create a GitHub Action workflow file 
nano  .github/workflows/run_build_tests.yml 
git commit -m "Added GitHub action: run current build tests"
git push origin $FEATURE_BRANCH
```
**.github/workflows/run_build_tests.yml**  
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

## Add a job to an existing GHA Workflow to build and push artifact to registry
In the next step, we will modify .github/workflows/run_build_tests.yml to add a new job that:  
* in case of successful test results builds a Docker image (artifact) that runs the branch code (a python web service)
* and push the image to DockerHub (artifact registry).


```shell
# Step 1. Create a feature branch
# Create a GitHub Action workflow file 
nano  .github/workflows/run_build_tests.yml 
git commit -m "Added GitHub action: run current build tests"
git push origin $FEATURE_BRANCH
``` 

**.github/workflows/run_build_tests.yml**  

*Note: Showing partial view of unittests job. Check full code here:* [py-holder](https://github.com/amesones-dev/py-holder.git) 


```console
name: Run automated unit tests and build new-feature
on:
  push:
    branches:
      - '**'
      - '!main'

env:
  TEST_NOT_OK: NOT_OK

jobs:
  unittests:
    runs-on: ubuntu-latest
    ...
    outputs:
      test-result: ${{ steps.test-report.outputs.result }}
    ...
    - id: test-report
      name: Generate test result outputs
      run: |
            ...
            echo "result=${result}" >> $GITHUB_OUTPUT

  branch_build_and_push:
    runs-on: ubuntu-latest
    needs: unittests
    outputs:
      build-docker-tag: ${{ steps.tag-push.outputs.result }}
    env:
      TEST_ERR_MSG: "Unittests failed. Build job cannot continue"
      TEST_OK_MSG: "Unittests OK. Proceeding to build job"

      BUILD_DOCKERFILE: ./run/Dockerfile
      BUILD_DOCKER_TAG: build-${GITHUB_REPOSITORY_ID}-${GITHUB_REF_NAME}-$GITHUB_RUN_ID
      BUILD_LOG: build-${GITHUB_RUN_ID}-result.log
    steps:
      - name: Inspect tests result
        run: |
            echo "Tests result: ${{needs.unittests.outputs.test-result}}"

      - name: Check tests result
        # Do not use ${{ }} with if conditions
        # Reference: https://docs.github.com/en/actions/using-jobs/using-conditions-to-control-job-execution
        if: needs.unittests.outputs.test-result==env.TEST_NOT_OK
        run: |
            exit 1

      - uses: actions/checkout@v3

      - name: Build
        if: success()
        run:  | 
              echo ${{ env.TEST_OK_MSG }}
              docker build . --file  ${{ env.BUILD_DOCKERFILE }}  --tag ${{ env.BUILD_DOCKER_TAG }}

      - name: Log in to Docker Hub
        if: success()
        uses: docker/login-action@f4ef78c080cd8ba55a85445d5b36e214a81df20a
        with:
          username: ${{ secrets.DOCKERHUB_USER }}
          password: ${{ secrets.DOCKERHUB_PASSWORD }}

      - name: Tag and push
        id: tag-push
        if: success()
        run: |
          export REMOTE_TAG="${{ vars.DOCKERHUB_REPO }}:${{ env.BUILD_DOCKER_TAG }}"
          docker tag ${{ env.BUILD_DOCKER_TAG }} ${REMOTE_TAG}
          docker push ${REMOTE_TAG}
          result=${{ env.BUILD_DOCKER_TAG }}
          echo "result=${result}"
          echo "result=${result}" >> $GITHUB_OUTPUT  
```

**Notes on GHA Workflow**

* The action triggers on [push](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#push) for
all branches except main, following
[branch trigger filtering rules](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions#onpull_requestpull_request_targetbranchesbranches-ignore).  

* Runs 2 jobs: **unittets** and **branch_build_and_push**
  * **branch_build_and_push** depends on **unittets**
  * **branch_build_and_push** uses a **unittets** output, called test-result

* **unittests** builds and runs a docker image to run python unit tests as per branch code  
* **branch_build_and_push** builds the application docker image if the tests finish without errors
* **branch_build_and_push** generates and output to be used by other jobs or workflows with the built docker image tag

**Notes on new GHA features**
* The env variable *TEST_NOT_OK* has been promoted to workflow scope because it is used by the two jobs
* **branch_build_and_push** step **Inspect tests result** uses the output test-result declared in **unittests** 
* **branch_build_and_push** step **Check tests result**  uses a conditional step execution **if**
* **branch_build_and_push** step **Log in to Docker Hub**  uses a predefined Action to log in to DockerHub
* **branch_build_and_push** step **Log in to Docker Hub**  uses GHA  secrets to log in to DockerHub

## Add a job to pull artifact from registry and run application Smoke Tests
The goal of this job is to inspect how further CI procedures will consume the artifact generated in the previous example.



**About conditional execution with if and variables**  
[Reference] (https://docs.github.com/en/actions/using-jobs/using-conditions-to-control-job-execution)

**About user environment variables**  

GHA provides different kinds of [variables](https://docs.github.com/en/actions/learn-github-actions/variables):
* Predefined environment variables
* User defined environment variables to be used by one or more workflows.

In the example, only user environment variables for a single workflow are used. 
Within a workflow, environment variables:  
* can have different 
[scopes](https://docs.github.com/en/actions/learn-github-actions/variables#defining-environment-variables-for-a-single-workflow): workflow, job, step.
* can be accessed as ${{ env.NAME }} within their scope


**About Job Outputs**  
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

**About Predefined actions**  
[Predefined actions](https://docs.github.com/en/actions/creating-actions/about-custom-actions)
