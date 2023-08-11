---
layout: post
title:  "CI basics: creating build from a git branch (1/2)"
date:   2023-08-09
categories: jekyll update
---
Audience: 
* You want to start using CI/CD procedures in Google Cloud Platform.
* You are familiar with CI procedures.
* You are familiar with gcloud, git,  docker, python.
* You want to start coding *without any costs* for now. 

## Creating a docker artifact from a specific git repo branch
### CI procedures introduction  
In this guide, we will introduce the foundations of 
[Continuous Integration](https://cloud.google.com/architecture/devops/devops-tech-continuous-integration)
and inspect a basic procedure leading to automating the process of building, testing, and delivering artifacts:
*creating a docker artifact from a specific git repo branch*.
 
**CI building blocks summary**
To successfully apply CI to a software delivery system, the following processes must be in place:
1. An automated build process.
   * Script that run builds and create artifacts that can be deployed to any environment.  
   * Builds can de identified, referenced and repeatable.
   * Frequent builds  
2. A suite of automated tests that must be successful as condition for artifact creation.
   * Unit tests
   * Acceptance tests
3. A CI system that runs the build and automated tests for every new version of code.
4. Small and frequent code updates to trunk-based developments, usually implemented with tools like Git and Git based 
products like [GitHub](https://github.com/), [BitBucket](https://bitbucket.org/), 
 [Cloud Repositories](https://cloud.google.com/source-repositories/docs), etc., where code versions are organized in one
or several environments with a main branch and feature branches that developers check out, modify and, after 
automatically testing and passing QA tests, merge to original branches via pull requests.  
5. An agreement that when the build breaks, fixing it should take priority over any other work.  

This demo will use basic tools like docker and shell, since the goal is to inspect the CI process itself, without 
focusing on a particular Cloud platform solution (no GCP costs involved, the example can be run in a GCP project
without bill enabled).  

Google Cloud has their own set of CI/CD tools, that will be considered in future posts.
* [Cloud Build](https://cloud.google.com/build/docs)
* [Cloud Deploy](https://cloud.google.com/deploy/docs)
* [Cloud Repositories](https://cloud.google.com/source-repositories/docs)
* [Artifact Registry](https://cloud.google.com/artifact-registry/docs)

 
**References**
* About [Continuous Integration](https://cloud.google.com/architecture/devops/devops-tech-continuous-integration)
by Google
* [CI/CD quickstart](https://cloud.google.com/docs/ci-cd) by Google
* [Devops CI](https://cloud.google.com/architecture/devops/devops-tech-continuous-integration) by Google 
 

### Local environment build of a specific feature branch
* The example uses the repo [gfs-log-manager](https://github.com/amesones-dev/gfs-log-manager.git).  
* The [ci_procs](https://github.com/amesones-dev/gfs-log-manager/tree/ci_procs) branch contains a
[Dockerfile](https://github.com/amesones-dev/gfs-log-manager/blob/ci_procs/run/Dockerfile) to build 
and run the application with docker engine.  
* A running docker based on the Dockerfile calls python to run the application with 
[Flask](https://flask.palletsprojects.com/) as per [start.py](https://github.com/amesones-dev/gfs-log-manager/blob/ci_procs/src/start.py)

#### Run code from Cloud Shell
1. Create a [Google Cloud](https://console.cloud.google.com/home/dashboard) platform account if you do not already have it.
2. Create a [Google Cloud project](https://developers.google.com/workspace/guides/create-project) or use an existing one.
3. Launch [Google Cloud Shell](https://console.cloud.google.com/home/)

#### Clone repo and checkout specific branch
In automated CI systems, the repo and branch are provided as input to the automated building process. 
```shell
# Local build
REPO='https://github.com/amesones-dev/gfs-log-manager.git'
REPO_NAME='gfs-log-manager'
export FEATURE_BRANCH="ci_procs"

git clone ${REPO}
cd ${REPO_NAME}

# Select branch. Ideally use a specific convention for branch naming
export FEATURE_BRANCH="ci_procs"
# Check that the branch exists
git branch -a |grep ${FEATURE_BRANCH}

git checkout ${FEATURE_BRANCH}
# Output
    branch 'ci_procs' set up to track 'origin/ci_procs'.
    Switched to a new branch 'ci_procs'
````    

#### Build Dockerfile stored in feature branch
[Inspect Dockerfile](https://raw.githubusercontent.com/amesones-dev/gfs-log-manager/ci_procs/run/Dockerfile)
```shell
# Identify your build
# Usually automated CI systems provide UUID for build IDs and maintains a Builds database
export BUILD_ID=$(python -c "import uuid;print(uuid.uuid4())")

# Use a meaningful local docker image tag
# Automated CI systems can generate a docker image tag for you
export RID="${RANDOM}-$(date +%s)" 
export LOCAL_DOCKER_IMG_TAG="${REPO_NAME}-${FEATURE_BRANCH}-${RID}"

# Launch build process with docker
# The build is done with your local environment docker engine, build logs captured
docker build . -f ./run/Dockerfile -t ${LOCAL_DOCKER_IMG_TAG} --no-cache --progress=plain  2>&1 | tee ${BUILD_ID}.log
```

**About builds, artifacts and build management systems**  

The example is a simple one-step build. However, in general, during CI procedures:
* A build can have **many build steps** and generate **several artifacts** of different kinds (Docker images, 
 Kubernetes configurations, Helm charts, etc.).
* Builds can be off-loaded to remote systems that implement an automated build engine API
* Build IDs, environments,  configurations, logs and results are stored in Build management systems so builds can be 
inspected, analyzed and repeated if needed.
* Artifacts are usually stored independently of builds and, in that case, Build management systems store references 
to every artifact generated during the build
  
  

**Inspect build and artifacts details**
```shell
echo $BUILD_ID
# Output 
  45e4b913-dc76-4aa9-9898-217490fdd0fd

tail -n 5 "${BUILD_ID}.log"
# Output
    # 10 exporting layers
    #10 exporting layers 0.8s done
    #10 writing image sha256:a0bdb9a4065cd834549d1c2c7586c96005c1d05f0e1732e5f13edc715d62cd2b done
    #10 naming to docker.io/library/gfs-log-manager-ci_procs-24754-1691654416 done
  #10 DONE 0.8s

head -n 5 "${BUILD_ID}.log"
# Output
    # 0 building with "default" instance using docker driver
    
    #1 [internal] load build definition from Dockerfile
    #1 transferring dockerfile: 502B done
    #1 DONE 0.0s
    
    
# Artifact (docker image) details
docker image ls ${LOCAL_DOCKER_IMG_TAG}
# Output
  REPOSITORY                                  TAG       IMAGE ID       CREATED         SIZE
  gfs-log-manager-ci_procs-24754-1691654416   latest    a0bdb9a4065c   5 seconds ago   102MB
     
```

#### Run the newly built docker image
* Set container port for running application  
 
```shell
# Default container port is 8080 if PORT not specified
export PORT=8081
```

* Set the local environment for the running docker image  
Usually applications expect a number of config values to be present in the running environment as variables.  
  * The demo app expects as minimal configuration the location for the Service Account(SA) key file that will identify 
  the app when accessing Cloud Logging API.  
  * Check ["Access to GCP resources"](/blog/jekyll/update/2023/08/01/k8s-learn-on-gcp) and the demo app the repo 
[README](https://github.com/amesones-dev/gfs-log-manager#readme)  for more information.
  * The engine running the python application, Flask, needs a secret key for randomizing sessions, etc.
 

```shell
export LG_SA_KEY_JSON_FILE='/etc/secrets/sa_key_lg.json'
export FLASK_SECRET_KEY =$(openssl rand -base64 128)
```

* Run the docker image  
 
```shell
# Known local path containing  SA key sa_key_lg.json
export LOCAL_SA_KEY_PATH='/secure_location'

# Set environment with -e
# Publish app port with -p 
# Mount LOCAL_SA_KEY_PATH to '/etc/secrets' in running container
docker run -e PORT -e LG_SA_KEY_JSON_FILE -e FLASK_SECRET_KEY -p ${PORT}:${PORT}  -v "${LOCAL_SA_KEY_PATH}":/etc/secrets  ${LOCAL_DOCKER_IMG_TAG}
```

### Watch the app running with  Web Preview
In the example, launch Web Preview on Cloud Shell, setting port to 8081.



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





