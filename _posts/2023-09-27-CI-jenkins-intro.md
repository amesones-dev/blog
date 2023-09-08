---
layout: post
title:  "CI/CD: Setting up Jenkins in K8S with Helm"
date:   2023-09-27
categories: jekyll update
---

Audience: 
* You want to start using CI/CD procedures in Google Cloud Platform.
* You are familiar with CI procedures.
* You are familiar with gcloud, git,  docker, python, gh, GitHub, jenkins
* You want to start coding *without any costs* for now. 

In this guide, we will 
* set up Jenkins on minikube using Helm chart
* configure Jenkins to spawn agents on Kubernetes to run CI/CD procedures
* both as a requirement to set up a basic Jenkins pipeline for a python app
* the pipeline code itself will be explored on an upcoming post 

## Jenkins setup with Helm chart on K8S
In this guide, you will revisit the environment set up in 
[Exploring Helm for agile K8S apps installation](/blog/jekyll/update/2023/08/04/helm-intro)
to run the Jenkins server that will manage a CI/CD pipeline. The goal is to prepare a jenkins server to manage
a CI/CD pipeline for python applications. 
Once the jenkins environment is prepared, we will explore how to create and run the pipeline for a python code 
repository in a new post.

### Set up K8S environment

If you do not have access to a Kubernetes cluster, like [GKE](https://cloud.google.com/kubernetes-engine),  you can run 
minikube in [Google Cloud Shell](https://console.cloud.google.com/home/) or use a local environment to set
it up.  
Configure K8S cluster using your available Kubernetes environment.

#### minikube
```shell
# Create minikube cluster
# Start K8S cluster with minikube
minikube start
minikube dashboard &
minikube tunnel &
kubectl config current-context
kubectl cluster-info


```
#### GKE
If you have access to an existing GKE cluster, configure kubectl to use an existing GKE cluster.
In that case there is a **cost** in running the example.
```shell
# Instructions to create a GKE cluster
export PROJECT_ID="YOUR_GCP_PROJECT"
export CLUSTER="YOUR_CLUSTER_NAME"
export ZONE=europe-west2-b
export REGION=europe-west2
export CLUSTER_NUM_NODES=3

# Requirements
APIS="container.googleapis.com cloudbuild.googleapis.com"
for API in ${APIS} 
do
	gcloud services enable ${API}
done

# Bootstrap
gcloud config set project ${PROJECT_ID}
gcloud config set compute/region ${REGION}
gcloud config set compute/zone ${ZONE}

# Create  cluster with default configuration
# Default NUM_NODES=3, MACHINE_TYPE=E2-MEDIUM, default VPC, no HA (control plane and nodes in same zone zone) 
gcloud container clusters create $CLUSTER --async

# Alternatively use num-nodes to change nodes per zone value to a lower number for testing 
gcloud container clusters create $CLUSTER --num-nodes $CLUSTER_NUM_NODES --async

# Instructions to connect to an existing GKE cluster
gcloud container clusters list
export K8S="YOUR_CLUSTER_NAME"
gcloud container  ${K8S} get-credentials

# Connect with kubectl
kubectl config current-context
kubectl cluster-info
```
### Install Helm and Jenkins chart
If needed, quickly revisit [Exploring Helm for agile K8S apps installation](/blog/jekyll/update/2023/08/04/helm-intro) 
to refresh Helm concepts: charts, repositories, etc.  
#### Installing Helm
* Reference: [Helm Installation guide](https://helm.sh/docs/intro/install/)
```shell
# Check current URL for latest version
export HELM_SCRIPT_URL=https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
curl -fsSL -o get_helm.sh ${HELM_SCRIPT_URL}
chmod 700 get_helm.sh

# Install helm to your current kubectl cluster
./get_helm.sh

# Check installation
helm version
```

#### Adding Jenkins Helm repository
```shell
# Repo management
#Adding a repository
export REPO_NAME=jenkins 
export REPO_URL=https://charts.jenkins.io
helm repo add ${REPO_NAME} ${REPO_URL}
helm repo update
# List charts in repositories you have already added
helm search repo 

# Search a chart in your current repo
export SEARCH_STRING=jenkins
helm search repo ${SEARCH_STRING}
# Output (version numbers might differ)
# jenkins/jenkins 4.6.5     2.414.2         Jenkins - Build 
```
#### Installing Jenkins chart
```shell
# Installing jenkins
export CHART=jenkins/jenkins

# Getting chart information
helm inspect chart ${CHART}
# Output
#...
# description: Jenkins - Build great things at any scale! The leading open source automation
# server, Jenkins provides over 1800 plugins to support building, deploying and automating
# any project.
# home: https://jenkins.io/
#  icon: https://get.jenkins.io/art/jenkins-logo/logo.svg
# ...

# Download Helm chart
helm pull ${CHART}
ls   jenkins*.tgz
# Note the version number of the compressed chart and extract
tar -xf jenkins-4.6.5.tgz

# Chart customizable settings
# Inspect values.yaml and modify if needed
# jenkins/values.yaml
cp jenkins/values.yaml ~/values.yaml
nano values.yaml

# Set Helm release name 
export RID=${RANDOM}
export RELEASE_NAME="jenkins-${RID}"
# Set K8S cluster namespace
export NAMESPACE=dev

# Deploying chart with standard values
helm install ${CHART} --name-template ${RELEASE_NAME}  --namespace=${NAMESPACE} --create-namespace 

# Or with modifications using local edited copy of values.yaml
helm install ${CHART} --name-template ${RELEASE_NAME}  --namespace=${NAMESPACE} --create-namespace  -f ~/values.yaml
```

**Inspect the Helm release details**  
```console
NAME: jenkins-10179
LAST DEPLOYED: Wed Sep 27 09:47:33 2023
NAMESPACE: dev
STATUS: deployed
REVISION: 1
NOTES:
1. Get your 'admin' user password by running:
  kubectl exec --namespace dev -it svc/jenkins-10179 -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
  echo http://127.0.0.1:8080
  kubectl --namespace dev port-forward svc/jenkins-10179 8080:8080

3. Login with the password from step 1 and the username: admin
4. Configure security realm and authorization strategy
5. Use Jenkins Configuration as Code by specifying configScripts in your values.yaml file, see documentation: http://127.0.0.1:8080/configuration-as-code and examples: https://github.com/jenkinsci/configuration-as-code-plugin/tree/master/demos

For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine

For more information about Jenkins Configuration as Code, visit:
https://jenkins.io/projects/jcasc/


NOTE: Consider using a custom image with pre-installed plugins
```
**Note K8S objects deployed by the Helm chart**
```shell
kubectl get services --namespace dev
# Output
# NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
# jenkins-10179         ClusterIP   10.100.2.143    <none>        8080/TCP    5m52s
# jenkins-10179-agent   ClusterIP   10.110.192.21   <none>        50000/TCP   5m52s

kubectl get statefulsets --namespace dev
# NAME            READY   AGE
# jenkins-10179   1/1     7m7s

kubectl get pods  --namespace dev
# NAME              READY   STATUS    RESTARTS   AGE
# jenkins-10179-0   2/2     Running   0          7m11s

kubectl get secrets  --namespace dev
# NAME                                  TYPE                 DATA   AGE
# jenkins-10179                         Opaque               2      7m43s
# sh.helm.release.v1.jenkins-10179.v1   helm.sh/release.v1   1      7m43s

kubectl get configmaps  --namespace dev
# NAME                                 DATA   AGE
# jenkins-10179                        2      7m50s
# jenkins-10179-jenkins-jcasc-config   1      7m50s
# kube-root-ca.crt                     1      7m50s
```

#### Get jenkins admin password
```
# Obtain Jenkins admin password

# Method 1
kubectl exec --namespace dev -it svc/jenkins-${RID} -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
****************
# Method 2
echo $(kubectl get secret --namespace dev jenkins-${RID} -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode)
# Output
# Password: ****************
```
#### Access  jenkins admin site
By default, Jenkins service is published as a ClusterIP K8S service.
```
kubectl get service jenkins-$RID  --namespace dev
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
jenkins-10179   ClusterIP   10.100.2.143   <none>        8080/TCP   18m
```

To access the jenkins admin site you need to make the service available, so it can be accessed with a browser.
This can be achieved by forwarding the service to a reachable localhost address and port.
```
# Forward K8S service to a browsable localhost port  
kubectl --namespace dev port-forward svc/jenkins-${RID} 8080:8080
```
If running on Cloud Shell, use [Cloud Shell Web Preview](https://cloud.google.com/shell/docs/using-web-preview) on port 
8080 after forwarding service port to get access to Jenkins admin site, and log in with the admin password. 

Once browsing jenkins admin site, login with username **admin** and password obtained as described.  

![jenkins admin login ](/blog/res/img/jenkins-login.jpg)  
  
*Modifying Jenkins service type*

If using GKE and planning to keep the Jenkins service, you can choose to publish the Jenkins service to a public IP by 
changing the service type to LoadBalancer, but there's a **cost involved** due to the use of a public IP in GCP.
It can be done by modifying value.yaml before deployment changing the serviceType for the chart.
```shell
grep serviceType  values.yaml
# Output 
#  serviceType: ClusterIP
sed -i  "s#serviceType: ClusterIP#serviceType: LoadBalancer#g" values.yaml
grep serviceType  values.yaml
# Output 
#  serviceType: LoadBalancer

helm install ${CHART} --name-template ${RELEASE_NAME}  --namespace=${NAMESPACE} --create-namespace  -f ~/values.yaml 
```

### Configure Jenkins  
#### Configure Kubernetes plugin  
The Helm chart automatically installs the Kubernetes plugin for Jenkins. The list of jenkins plugins to be installed 
can be customized in the chart configuration values.yaml
From the Jenkins admin site go to:
* Dashboard, Manage Jenkins, Clouds  
* Select 'kubernetes'  
* Click on Configure
* Select Cloud Kubernetes configuration
* Expand Kubernetes Cloud Details

 
  ![jenkins clouds](/blog/res/img/jenkins-cloud.png)

* Check the following values match namespace and helm release(*dev and 10179 in example*)
  * Kubernetes URL: https://kubernetes.default.svc.cluster.local
  * Jenkins URL: http://jenkins-10179.dev.svc.cluster.local:8080
  * Press the **Test Connection** to check Jenkins can connect to local cluster: it should display 
  *Connected to Kubernetes v1.27.4*, with the version number corresponding to minikube K8S version.
 ![Cloud kubernetes configuration](/blog/res/img/jenkins-cloud-config.png)

#### K8S jenkins agents configuration
  When using a Kubernetes cluster to run build agents Jenkins will launch pods using the Pod template specified in the 
Cloud configuration. By default, these pods run one single container with a jenkins agent container image. 
  
  * Notice the section **Pod Templates, Pod Template details**  
  * Check that namespace and label match  *dev* and *jenkins/jenkins-10179-jenkins-agent*
  * The pods are labelled so the jenkins administrator can assign the agents to specific build jobs if needed. 
  New labels cand be added if so wished.  
  * In the same section, under Container, Container image, check that the image name is set to 
[jenkins/inbound-agent](https://hub.docker.com/r/jenkins/inbound-agent/)
  * Command and arguments should be empty. The image entrypoint runs by default a jenkins agent that connects to Jenkins
  and register the agent as available for running jobs.
  * A jenkins admin can customize the pod template adding other containers to the pod to execute a specific kind of jobs.


```yaml
# jenkins Pod agent example
---
apiVersion: "v1"
kind: "Pod"
metadata:
  labels:
    jenkins/jenkins-10179-jenkins-agent: "true"
    jenkins/label-digest: "91075ff972a9ed5453b9d64a77e61bb5e529bffd"
    jenkins/label: "jenkins-10179-jenkins-agent"
  name: "default-15dtz"
  namespace: "dev"
spec:
  containers:
  - env:
    - name: "JENKINS_SECRET"
      value: "********"
    - name: "JENKINS_TUNNEL"
      value: "jenkins-10179-agent.dev.svc.cluster.local:50000"
    - name: "JENKINS_AGENT_NAME"
      value: "default-15dtz"
    - name: "JENKINS_NAME"
      value: "default-15dtz"
    - name: "JENKINS_AGENT_WORKDIR"
      value: "/home/jenkins/agent"
    - name: "JENKINS_URL"
      value: "http://jenkins-10179.dev.svc.cluster.local:8080/"
    image: "jenkins/inbound-agent"
    imagePullPolicy: "IfNotPresent"
    name: "jnlp"
    resources:
      limits:
        memory: "512Mi"
        cpu: "512m"
      requests:
        memory: "512Mi"
        cpu: "512m"
    tty: false
    volumeMounts:
    - mountPath: "/home/jenkins/agent"
      name: "workspace-volume"
      readOnly: false
    workingDir: "/home/jenkins/agent"
  hostNetwork: false
  nodeSelector:
    kubernetes.io/os: "linux"
  restartPolicy: "Never"
  serviceAccountName: "default"
  volumes:
  - emptyDir:
      medium: ""
    name: "workspace-volume"

```


#### Create a test FreeStyle Project
Finally, once configured Kubernetes as a jenkins agent source, you can create a simple pipeline to test it.

**Create Test build**  
* Go to Jenkins, New Item, Free Style Project
* Choose a Name: *K8S Test*
* General, Restrict where this project can be run
  * Label Expression: **jenkins-10179-jenkins-agent**  
* Build, Add Step, Execute shell
  * Command: Add a simple sequence of commands, like this one below.
```shell
echo "K8S Test OK"
printenv
```
**Launch a build**
* Go to Jenkins, Dashboard, K8S Test, click on Build Now
![jenkins build](/blog/res/img/jenkins-cloud-build.png) 


* Look at the Build history and click on #1 and then Console Output to observe the build steps log
```shell
Building remotely on default-0kn6l (jenkins-10179-jenkins-agent) in workspace /home/jenkins/agent/workspace/K8S Test
[K8S Test] $ /bin/sh -xe /tmp/jenkins2197527110963357658.sh
+ echo K8S test
K8S test
+ printenv
JENKINS_HOME=/var/jenkins_home
KUBERNETES_PORT=tcp://10.96.0.1:443
...
JENKINS_NAME=default-0kn6l
JOB_NAME=K8S Test
JENKINS_10179_PORT=tcp://10.97.176.101:8080
PWD=/home/jenkins/agent/workspace/K8S Test
JAVA_HOME=/opt/java/openjdk
JENKINS_10179_SERVICE_PORT=8080
KUBERNETES_SERVICE_HOST=10.96.0.1
...
Finished: SUCCESS
 ```
 
### Next time
Once Jenkins is set up on a K8S cluster and can run Pod agents, we will explore how to code a jenkins CI pipeline for a 
python application with similar functionality to the GitHub Actions pipeline showcased in 
[CI: Automating builds with GitHub Actions](/blog/jekyll/update/2023/09/04/CI-auto-build)
