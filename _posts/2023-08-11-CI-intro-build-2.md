---
layout: post
title:  "CI basics: creating build from a git branch (2/2)"
date:   2023-08-11
categories: jekyll update
---
Audience: 
* You want to start using CI/CD procedures in Google Cloud Platform.
* You are familiar with CI procedures.
* You are familiar with gcloud, git,  docker, python.
* You want to start coding *without any costs* for now. 

## Creating a docker artifact from a specific git repo branch
Following  last post [CI basics: creating build from a git branch (1/2)](/blog/jekyll/update/2023/08/09/CI-intro-build), 
in this guide we will complete the example script  whilst exploring related   
[CI](https://cloud.google.com/architecture/devops/devops-tech-continuous-integration) principles.

The example script is a minimal implementation of a build that creates artifacts, which is the first step needed to 
create an automated build process, one of CI building blocks. 
* At the moment, the script is a manual procedure, so we can inspect the several steps and relate them to CI principles.
* We will later inspect how to make it automatic and in sync with changes to the git branch code created by code developers.


### Previously...
We showed how a CI automated systems would provide the inputs for the build process: repo and branch information 
```shell
REPO='https://github.com/amesones-dev/gfs-log-manager.git'
REPO_NAME='gfs-log-manage
export FEATURE_BRANCH="ci_procs"

```
The build process would :
* clone the repo and checkout specific branch
* identity the build to maintain a Builds database
* launch, execute and collect build results: status, logs, artifacts
* identify the generated artifacts to stored them in artifact registries and associate them to specific builds

```shell
export BUILD_ID=$(python -c "import uuid;print(uuid.uuid4())")
export LOCAL_DOCKER_IMG_TAG="${REPO_NAME}-${FEATURE_BRANCH}-${RID}"
...
```

The last step we inspected was how to check the generated artifacts. In this case, as an example, the single-step build 
produces a single docker image, that can be run with docker engine.
The docker was up and running and could be inspected with Web Preview on Cloud Shell.
```shell
...
docker run -e PORT -e LG_SA_KEY_JSON_FILE -e FLASK_SECRET_KEY -p ${PORT}:${PORT}  -v "${LOCAL_SA_KEY_PATH}":/etc/secrets  ${LOCAL_DOCKER_IMG_TAG}
```

**Basic check**  
Before we introduce the concept of testing in CI procedures, these  few commands check that the running docker image is 
working as expected.

```shell
# Basic app tests
# Main url
curl --head  localhost:8081
# Output
  HTTP/1.1 200 OK
  Content-Type: text/html; charset=utf-8
  ...
  
# The app implements a /healthcheck endpoint that can be used for liveness and readiness probes
# Output set by app design
curl -i localhost:8081/healthcheck
  HTTP/1.1 200 OK
  ...
  Content-Type: application/json

  {"status":"OK"}

# Test any app endpoints as needed
export ENDPOINT='index'
curl -I  localhost:8081/${ENDPOINT}
# Output 
  HTTP/1.1 200 OK
  ...
 
curl -I -s  localhost:8081/${ENDPOINT} --output http-test-${ENDPOINT}.log
grep   'HTTP' http-test-${ENDPOINT}.log
# Output
  HTTP/1.1 200 OK

# Non existent endpoint
export ENDPOINT=app_does_not_implement
curl -I -s  localhost:8081/${ENDPOINT} --output http-test-${ENDPOINT}.log 
grep   'HTTP' http-test-${ENDPOINT}.log
# Output
  HTTP/1.1 404 NOT FOUND
   
```


### About testing the build
So far, there have not been any execution errors (it can be checked with the build logs). But, is this enough to 
consider the build successful?

* Part of the CI process involves designing a number of conditions or checkpoints for the build products to be 
considered a valid input for successive CI processes. Generally speaking, tests are procedures that check that these 
conditions are met.  
* In the developing world, the word *'test'* has a specific meaning and usually refers to pieces of code included with 
the branch code itself, that use well-known testing methods and libraries and are designed to be used as input to 
standard and ideally automated testing procedures. There are code integrated tests of several kinds that apply to 
different phases of the build process in particular and the CI process in general.

**Test, what test?**
In a fast-paced development environment developers are constantly updating code. A change of the branch code  can break
the build in  several  ways:
* The build system cannot fetch the code 
* The build  does not run successfully
* The build runs but the application does not respond
* The build runs but application does not do what is supposed to do.
* etc.
There are tests that check most of these scenarios during the CI process. 
For a detailed look, check the reference links below.

*Reference*  

[Simple code-integrated test example](https://github.com/amesones-dev/gfs-log-manager/blob/main/tests/tests.py)  
[CI testings](https://www.harness.io/blog/continuous-integration-testing) by Harness  
[Testing methods](https://circleci.com/blog/testing-methods-all-developers-should-know/) by CircleCI  


#### Making tests an integral part of the build  
Since tests can be coded and are required to check that a build is successful, the next logical step is making the tests
an integral part of the code and the build process. Code designed for CI often includes along the source code a number 
of tests that can be integrated into the build process.

*Introducing testing*
```shell
# Build inputs
REPO='https://github.com/amesones-dev/gfs-log-manager.git'
REPO_NAME='gfs-log-manager'
export FEATURE_BRANCH="ci_procs"

# Provide Build ID and identify generated artifacts
export BUILD_ID=$(python -c "import uuid;print(uuid.uuid4())")
export RID="${RANDOM}-$(date +%s)" 
export LOCAL_DOCKER_IMG_TAG="${REPO_NAME}-${FEATURE_BRANCH}-${RID}"

# Provide Test ID and identify generated artifacts
export TID=$(python -c "import uuid;print(uuid.uuid4())")
export LOCAL_DOCKER_IMG_TAG_TEST="test-${LOCAL_DOCKER_IMG_TAG}"

# Build process

# Phase 0. Checkout branch
git clone ${REPO}
cd ${REPO_NAME}
git checkout ${FEATURE_BRANCH}



# Phase 1. Testing
# Create a docker image with the branch code that run the code built-in tests
# Generates a new artifact (docker image) that can be stored in an artifact registry as part of the build output    
docker build . -f ./run/Dockerfile-test   -t ${LOCAL_DOCKER_IMG_TAG_TEST}  --no-cache --progress=plain  2>&1 | tee ${TEST_ID}.log

# Run tests
# Run image and check that tests are successful. 
# You may want to use a different set of environment variables to run tests
# Alternatively, code built-in tests can use a specific configuration defined inline.
docker run -e LG_SA_KEY_JSON_FILE -e FLASK_SECRET_KEY -v "${LOCAL_SA_KEY_PATH}":/etc/secrets ${LOCAL_DOCKER_IMG_TAG_TEST} 2>&1 | tee ${TEST_ID}-result.log
# Output
  test_0_gl_creation_check (__main__.GlogManagerTestCase) ... ok
  ---------------------------------------------------------------------
  Ran 1 test in 0.262s
  OK
# Usually the output or logs are examined to determine success status
  grep 'OK' ""${TEST_ID}-result.log""    

# If the tests are not successful the overall Build process should fail and provide information
# CI is by nature iterative and uses detailed sub-processes feedback and registered status to avoid needless repetition.


   
# Phase 2. Build
docker build . -f ./run/Dockerfile -t ${LOCAL_DOCKER_IMG_TAG} --no-cache --progress=plain  2>&1 | tee ${BUILD_ID}.log

export PORT=8081
export LG_SA_KEY_JSON_FILE='/etc/secrets/sa_key_lg.json'
export FLASK_SECRET_KEY=$(openssl rand -base64 128) 
export LOCAL_SA_KEY_PATH='/secure_location'
docker run -e PORT -e LG_SA_KEY_JSON_FILE -e FLASK_SECRET_KEY -p ${PORT}:${PORT}  -v "${LOCAL_SA_KEY_PATH}":/etc/secrets  ${LOCAL_DOCKER_IMG_TAG}
```


[Dockerfile for tests image](https://raw.githubusercontent.com/amesones-dev/gfs-log-manager/ci_procs/run/Dockerfile-tests)
```
...
# Copy sources and tests into the container at /app
COPY src/. .
COPY tests/. .
...
# Run tests when the container launches
ENTRYPOINT ["python", "tests.py"]
...
```
 



**Follows on part 2/2:** 
#### Creating a docker artifact from a specific git repo branch (part 2 of 2)
* Testing the application
  * Defining endpoints
  * Checking responses
  * Running unittests in feature branch
  * Note about automated testing tools

* Managing builds and uploading artifact to artifact registry
  * Builds and artifacts
  * Builds management 
  * Uploading artifacts to artifacts registries
  * Registry tags vs local build tags
  * Managing artifacts

* About popular artifact registries
* Artifacts as inputs to CI procedures  





