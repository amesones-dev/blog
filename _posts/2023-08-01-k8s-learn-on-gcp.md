---
layout: post
title:  "Leveraging free GCP resources to learn Kubernetes"
date:   2023-08-01
categories: jekyll update
---
Audience: 
* You want to start using CI/CD procedures in Google Cloud Platform. 
* You are familiar with gcloud, minikube, kubectl.
* You want to start coding *without any costs* for now. 


## Using Google Cloud Shell and minikube to run a K8S cluster
### Leveraging free resources to learn Kubernetes
#### How it all begins
Although GCP has its own version of a managed Kubernetes service called [GKE](https://cloud.google.com/kubernetes-engine),
it is still useful to be able to run a minimal Kubernetes deployment for proof of concept, learning or just for fun.

In this guide, you'll set up a Kubernetes development environment based on minikube from Google Cloud Shell, learn how 
to deploy to Kubernetes and how to preview published Kubernetes services from your browser.

It focuses on minimal requirements, simple Google Cloud authentication and using only services that do not require 
billing enabled on the project.

1. Create a [Google Cloud](https://console.cloud.google.com/home/dashboard)  platform account if you do not already have it.
2. Create a [Google Cloud project](https://developers.google.com/workspace/guides/create-project) or use an existing one.
3. Launch [Google Cloud Shell](https://console.cloud.google.com/home/)

#### Install minikube in Cloud Shell
```console		   
> minikube start
   ...
    * Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default    
``` 

### Get a K8 dashboard going
1. Install the dashboard addon for minikube in a [new CloudShell tab](https://cloud.google.com/shell/docs/use-cloud-shell-terminal#open_multiple_terminal_sessions) 
```console	   
> minikube dashboard
        Verifying proxy health ...
        * Opening http://127.0.0.1:34649/api/v1/namespaces/kubernetes-dashboard...       
```

2. Change [Cloud Shell Web Preview](https://cloud.google.com/shell/docs/using-web-preview) port to the port number 
displayed in the above  output -*34649* in the example-  to see Kubernetes dashboard

3. Launch Web Preview to see Kubernetes API endpoints
```console
https://34649-cs-39836380922-default.cs-europe-west1-iuzs.cloudshell.dev/
```
4. Copy the Web Preview URL from your browser and add this endpoint to the root URL
```console
/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/
```

The full URL for Web Preview should look like this:  
```console 
https://34649-cs-39836380922-default.cs-europe-west1-iuzs.cloudshell.dev/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/
```

The leading number is the Web Preview port number, the rest of the host name depends on your Google Cloud resources 
default location. 


### Simple K8S deployment and K8S service examples
1. Create example deployment with a known public container image
```console
 kubectl create deployment hello-minikube --image=k8s.gcr.io/echoserver:1.4
```  
2. Create a K8S NodePort service using the example deployment
```console
 # echoserver:1.4 default container  port is 8080
kubectl expose deployment hello-minikube --type=NodePort --port=8080
```
3. In a new CloudShell tab  Publish service to the outer world 
```console
# Forward service output to  port 8081
kubectl port-forward service/hello-minikube 8081:8080 --namespace=default
```
4. Change cloud Web Preview to port 8081 to see the service response on your local browser  
  *Note*.Web Preview is only accessible to a developer logged with their Google Cloud credentials.
  

5. Alternatively,  create a K8S LoadBalancer service
```console
kubectl expose deployment hello-minikube --type=LoadBalancer --port=8080 --name=hello-minikube-lb
```
In minikube, load balancing can be simulated with [minikube tunnelling](https://minikube.sigs.k8s.io/docs/commands/tunnel/).
```console
 minikube tunnel &
 kubectl port-forward service/hello-minikube-lb 8081:8080
```

6. Check your services running in the Kubernetes dashboard.
 
[Services in Kubernetes dashboard]({{ site.baseurl }}/res/img/gcp-k8s-dash.jpg)



