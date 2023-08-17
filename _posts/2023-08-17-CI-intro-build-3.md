
layout: post
title:  "CI basics: storing builds and artifacts (3/3)"
date:   2023-08-17
categories: jekyll update
---

Audience: 
* You want to start using CI/CD procedures in Google Cloud Platform.
* You are familiar with CI procedures.
* You are familiar with gcloud, git,  docker, python.
* You want to start coding *without any costs* for now. 

## CI basics
Following  last post [CI basics: making tests an integral part of the build (2/3)](/blog/jekyll/update/2023/08/09/CI-intro-build-2), 
in this guide we will complete the example script  whilst exploring related 
[CI](https://cloud.google.com/architecture/devops/devops-tech-continuous-integration) principles. The example script is 
a minimal implementation of a build that creates artifacts from a repo branch, which is the first step needed to create 
an automated build process, one of CI building blocks. 
* At the moment, the script is a manual procedure and  runs basic building and integrated testing. We will finish exploring 
basic CI principles by using a basic artifact registry system and finally considering ways of triggering the build process 
everytime  there is a  change on a tracked git code branch.

## Previously...
We showed how a CI automated system would provide the inputs for the build process: repo and branch information, then 
the build process would :
* clone the repo and checkout specific branch
* run automated tests
* identify the build to maintain a Builds database
* launch, execute and collect results: tests results, status, logs, artifacts
* identify the generated artifacts to stored them in artifact registries and associate them to specific builds  

## Builds and artifacts
Artifacts can be defined, in general, as  objects produced by the build process. 

The example build script produces two artifacts everytime is run:
* A docker image used to run tests
* A docker image that run the application.
So far these images are stored only locally and hence lost when the local environment cease to exist.  

In a CI process, builds are repeated as many times as there are changes to the code by any developer working on 
the same branch. Every build produces a number of artifacts. If the build is successful, the artifacts can be used for
the next steps of the CI process, but for that to be possible, it is first necessary to identify them, associate them 
to the build instance, and save them to permanent storage.



## Artifact registries
All these features can be grouped and delivered together by a dedicated CI subsystem, usually called artifact registry. 
An artifact registry can be defined as **a set of artifact repositories** (hierarchical storage systems) plus the interface
to manage them.  

Artifact registries can manage repositories of one specific kind of artifacts or many kinds (Docker images, Linux 
packages, JARs, Python packages, Node packages, etc.)  

Typical operations of artifact registries are:
* Create/delete an artifact repository
* Store an artifact in one of the registry artifact repositories
* Recover an artifact from one of the registry artifact repositories
* Manage access to repositories

**Reference**

* [On artifact registries](https://www.jetbrains.com/teamcity/ci-cd-guide/concepts/artifact-repository/)


**Popular artifact repositories/registries/management tools**
Generic Artifact management systems
* [Google Artifact Registry](https://cloud.google.com/artifact-registry)
* [AWS CodeArtifact](https://aws.amazon.com/codeartifact/)
* [JFrog Artifactory](https://jfrog.com/artifactory)  
* [Sonatype Nexus](https://www.sonatype.com/products/sonatype-nexus-repository) 

Build your own
* [Archiva](https://archiva.apache.org/)


Specific artifact types, public artifact repositories, etc.

For Docker images
* [Dockerhub](https://hub.docker.com/)

For Helm charts
* [Helm charts](https://artifacthub.io/)

### Uploading artifacts to artifacts registries
In this section we will modify the original build script to upload the local docker images to a private docker 
repository in Dockerhub, which would play the part of artifact registry for our introductory example to CI procedures.  

*Quick note about Dockerhub*  

[Dockerhub](https://hub.docker.com/) is a public docker repository which also offers repository hosting. There's a free 
basic access plan that allows to manage a single Docker image repository to upload your own Docker images. This plan is 
enough for the purpose of this guide: an introduction to artifacts and artifact management.

* Registry tags vs local build tags *
So far our build process has generated and store locally two docker images with specific local tags associated to repo, 
branch name and an extra ID that identifies the specific build run.
```shell
export LOCAL_DOCKER_IMG_TAG="${REPO_NAME}-${FEATURE_BRANCH}-${RID}"
export LOCAL_DOCKER_IMG_TAG_TEST="test-${LOCAL_DOCKER_IMG_TAG}"
```

```shell
export DOCKERHUB_USER='YOUR_DOCKERHUB_USER'
export DOCKERHUB_REPO='YOUR_DOCKERIMAGE_REPO'
docker login "$DOCKERHUB_USER"

# Builds docker image
export LOCAL_TAG=${LOCAL_DOCKER_IMG_TAG}
export REMOTE_TAG="${DOCKERHUB_USER}/${DOCKERHUB_REPO}:${LOCAL_TAG}" 

# Add remote tag to the docker image currently tag as <LOCAL_DOCKER_IMG_TAG> 
docker tag "${LOCAL__TAG}" "${REMOTE_TAG}"

# Push the image to the docker repository using the full remote tag    
docker push "${DOCKERHUB_USER}/${DOCKERHUB_REPO}:${LOCAL_TAG}"


# Tests image
export LOCAL_TAG=${LOCAL_DOCKER_IMG_TAG_TEST}
export REMOTE_TAG="${DOCKERHUB_USER}/${DOCKERHUB_REPO}:${LOCAL_TAG}"

# Add remote tag to the docker image currently tag as <LOCAL_DOCKER_IMG_TAG> 
docker tag "${LOCAL_TAG}" "${REMOTE_TAG}"

# Push the image to the docker repository using the full remote tag    
docker push "${DOCKERHUB_USER}/${DOCKERHUB_REPO}:${LOCAL_TAG}"

```  

**How to recover artifacts from artifact registry**
* Go to Dockerhub console
* Check your images
* Notice name, size and artifact download links
![Dockerhub artifacts](/blog/res/img/ci-3-dockerhub.jpg)


```shell
# Reusing your artifacts in other processes
docker pull amesones/dockerhub:gfs-log-manager-ci_procs-4189-1692269044

```




 




## Builds management systems

## Managing artifacts  

### Artifacts as inputs to CI procedures  

## Build automation: trigger builds on changes to git branch



### Follows on part 3/3 
#### Creating a docker artifact from a specific git repo branch
*Managing builds and uploading artifact to artifact registry*
  * Builds and artifacts
  * Repository tags vs local build tags
  * Uploading artifacts to artifacts registries
  * Managing artifacts
  * About popular artifact registries
  * Artifacts as inputs to CI procedures
*Builds management systems* 

  
  
