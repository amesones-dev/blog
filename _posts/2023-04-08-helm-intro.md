---
layout: post
title:  "Agile K8S apps installation: Helm"
date:   2023-08-04
categories: jekyll update
---
Audience: 
* You want to start using CI/CD procedures in Google Cloud Platform.
* You are familiar with the Kubernetes ecosystem
* You are familiar with gcloud, minikube, kubectl, GKE
* You want to start coding *without any costs* for now.



## Using minikube in Cloud Shell to explore Helm 
#### Leveraging free resources to learn Kubernetes
### How do I begin?

In this guide, you will use the Kubernetes environment set in 
[Leveraging free GCP resources to learn Kubernetes](2023-08-01-k8s-learn-on-gcp.md) to inspect Helm.

```console
# From Google Cloud Shell 
# Create minikube cluster for quick local demo
minikube start
# In a new Cloud Shell Tab
minikube dashboard &
minikube tunnel &
```

**Available GKE cluster**  
Alternatively, if you have access to a Google Cloud Project with billing enabled or using the 
[Google Cloud free trial](https://console.cloud.google.com/freetrial) you can use an existing 
[GKE](https://cloud.google.com/kubernetes-engine) cluster.   
Also do not forget to check the 
[Cloud Operations Sandbox](https://github.com/GoogleCloudPlatform/cloud-ops-sandbox) for more GCP DevOps utilities.  
*Note: mind that if you are using GKE and not during the free trial there are costs involved when running this guide*

```console
# Google Cloud Shell
# Connect  to existing GKE cluster
gcloud container clusters list
gcloud container CLUSTER_NAME get-credentials
```

### What is Helm
In a few words, [Helm]() is a tool that allows to package, reuse and customize Kubernetes applications.



### Helm basic concepts
Helm installs *charts* into Kubernetes, creating a new *release* for each installation. To find new charts, you can 
search Helm chart *repositories*  
* A *chart* is a Helm package. It contains all the resource definitions necessary to run an application in a Kubernetes cluster.
* A *repository* is the place where charts can be collected and shared.  
* A *release* is an instance of a chart running in a Kubernetes cluster. One chart can often be installed many times
    into the same cluster. And each time it is installed, a new release with its own name is created. 

### Installing Helm
Now that the Kubernetes cluster is up and running, it's time to install Helm. 
The latest instructions are available at [Helm Installation guide](https://helm.sh/docs/intro/install/)

```console
# Check current URL for latest version
export HELM_SCRIPT_URL="https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3"
curl -fsSL -o get_helm.sh ${HELM_SCRIPT_URL}
chmod 700 get_helm.sh

# Install helm to your current kubectl cluster
./get_helm.sh

# Check installation was successful
helm version
```
### Managing repositories
To install helm charts, first you need to find them and fetch them from repositories.

*Adding a repository to helm*
```console
# Helm repo management
# Adding a repository
export REPO_NAME=bitnami 
export REPO_URL="https://charts.bitnami.com/bitnami"
helm repo add ${REPO_NAME} ${REPO_URL}

# And another one
export REPO_NAME=jenkins 
export REPO_URL="https://charts.jenkins.io"
helm repo add ${REPO_NAME} ${REPO_URL}

helm repo update
```

To find charts available in a repo, you can search the repo contents:
```console
# List charts in repositories you have already added
helm search repo

# Search a chart in your current added repos
export SEARCH_STRING=mysql
helm search repo ${SEARCH_STRING}
```

### Installing a chart
Installing a chart triggers creation of a release, a set of objects in your Kubernetes environment, whatever is 
specified in the chart. Mostly, a chart definition contains a number of yaml files that define deployments, services, 
stateful sets, roles, service accounts,and other Kubernetes objects.


*Suggestion*  
Before and after creating release:
* If using minikube, check Kubernetes objects using Kubernetes dashboard 
* If using GKE, inspect workloads from [Google Cloud Console, Kubernetes, Workloads](https://console.cloud.google.com/kubernetes/workload)


```console
# Quick worlkloads peek before release
kubectl get pods -A

# Installing mysql
export RELEASE_NAME=dev-rel-dql-001
export CHART="bitnami/mysql"
helm install ${RELEASE_NAME} ${CHART}

# Quick worlkloads peek after release
kubectl get pods -A
```

Before continuing, let's uninstall the demo release
```console
# Remove a specific release
helm uninstall ${RELEASE_NAME}

# Quick worlkloads peek after release uninstall
kubectl get pods -A 
``` 



### Exploring chart contents
So, what is exactly in a chart?

*Getting chart info*
```console
# Getting chart information
helm inspect chart ${CHART}
    # Output
    ... 
    maintainers:
    - name: VMware, Inc.
      url: https://github.com/bitnami/charts
    name: mysql
    sources:
    - https://github.com/bitnami/charts/tree/main/bitnami/mysql
    version: 9.10.9
    ...
```

You can check the chart contents whether browsing the chart  home address (chart info output) or downloading the chart 
contents for local inspection.

*Download chart contents*
```console
helm pull ${CHART}
ls mysql*
tar -xf mysql-<version>.tgz
# Inspect extracted contents 
ls -R mysql
# Note this file, it will be referenced in the next sections
# mysql/values.yaml
```

### Where the magic happens: chart parametrization
Chart authors can choose to use customizable parameters for the Kubernetes objects definitions in their charts.
If that is the case, the chart contains a file called values.yaml with the default values for these parameters.
During installation, helm uses values.yaml by default, unless you set an alternative file.


```console
# After downloading and extracting chart contents
# Get values.yaml
cp mysql/values.yaml custom-values.yaml

# Inspect and update with new parameter values
nano custom-values.yaml

# From Cloud Shell, use the Code editor for convenience
edit custom-values.yaml
 
# You can set mysql parameters
    # custom-values.yaml
        [client]
        port=3306
        
# Or Kubernetes API objects parameters 
    # custom-values.yaml
          resources:
             limits:
                cpu: 250m
                memory: 256Mi

# Installing mysql with custom parameters
export RELEASE_NAME=mysql-r101
helm install $RELEASE_NAME ${CHART} -f custom-values.yaml
```

###Make releases more CI/CD friendly
*Use helm random release names and use helm namespaces*  
```console

# Simulating different namespaces with a random ID
export RID=$RANDOM
export NAMESPACE="dev-$RID"

# Install a release with a Helm generated random name in a specific Kubernets namespace
helm install ${CHART} --generate-name  --namespace=${NAMESPACE} --create-namespace

# List releases for namespace
helm list --namespace ${NAMESPACE} -q

# Since the release name was generated, it's useful to have an env variable with the name
# Command to show the latest installed release
export RELEASE_NAME=$(helm list --namespace ${NAMESPACE} -q -r -m 1)

echo Helm release ${RELEASE_NAME} running at K8S namespace ${NAMESPACE}

# Check Kubernetes objects created in the namespace  
kubectl get all  -n $NAMESPACE

# Repeat the process and create a few releases more changing parameters if so wished

```
 
### Clean up your  environment
#### Uninstall releases
```console
# Quick list all releases
helm list -A --no-headers

# A bit of awk 
RELS_UNINSTALL=$(helm list -A --no-headers|awk -F ' ' '{print "helm uninstall " $1 " --namespace " $2}')

echo "$RELS_UNINSTALL"
    helm uninstall mysql-1691154562 --namespace dev-7470
    helm uninstall mysql-1691154566 --namespace dev-23231
    helm uninstall mysql-1691154572 --namespace dev-18968
    helm uninstall mysql-1691154580 --namespace dev-26991
    
eval "$RELS_UNINSTALL"
    release "mysql-1691154562" uninstalled
    release "mysql-1691154566" uninstalled
    release "mysql-1691154572" uninstalled
    ...
```
**If using minikube**
```console
# Stops but keep minikube cluster configuration in CloudShell  
minikube stop
# Delete minikube cluster configuration and destroys cluster if so wished
minikube delete
```

### More about helm


#### About finding repos
Charts are authored resources, so it's up to the authors to publish them in repos. Finding reliable public existing 
repos is not a trivial matter. There are some public listings and specific tools created just for that:
* [Helm information](https://helm.sh/)
* [Artifact Hub](https://artifacthub.io/packages/helm/artifact-hub/artifact-hub).
* [Helm charts](https://artifacthub.io/)  
 
Apart from using repos, there are several alternative  ways to 
[find available charts](https://helm.sh/docs/topics/chart_repository/).


That's all for now about helm. There is much more, and I am sure we will revisit the subject on future occasions.
