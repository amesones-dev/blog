---
layout: post
title:  "Access to GCP resources"
date:   2023-07-31
categories: jekyll update
---

## Application access to Google Cloud resources in Python applications
### Python Cloud Client libraries
The [Python Cloud Client Libraries](https://cloud.google.com/python/docs/reference?l=python) are designed to use 
[Application Default Credentials](https://cloud.google.com/docs/authentication/application-default-credentials) to authenticate
and access Google Cloud resources according to [IAM](https://cloud.google.com/iam) policies.

### Explicit authentication from code with IAM Service Account key
You can use a [Service Account(SA) key](https://cloud.google.com/iam/docs/keys-create-delete) JSON file to manage 
application access to Google Cloud resources by using specific client libraries methods in the code that accept 
credential files as parameters.
 

```code
# myapp.py
def explicit():
    #Import BigQuert client library
    from google.cloud import bigquery
    
    # Explicitly use service account credentials by specifying the private key file.
    client = bigquery.Client.from_service_account_json('service_account.json')
    
```     

### Implicit authentication using GOOGLE_APPLICATION_CREDENTIALS
Alternatively, you can use  client libraries methods without any credential parameters and set the variable
[GOOGLE_APPLICATION_CREDENTIALS](https://cloud.google.com/docs/authentication/application-default-credentials) in your
environment to define how to control access to Google Cloud resources.

```code
# myapp.py
def implicit_():
    #Import BigQuert client library
    from google.cloud import bigquery
    
    # Client Libraries use credentials according to GOOGLE_APPLICATION_CREDENTIALS 
    client = bigquery.Client()

```        

#### Using a Service Account credential file
To access Google Cloud Services from code as an SA, set the environment variable GOOGLE_APPLICATION_CREDENTIALS to the 
JSON key file path and then run your code using Google services.  

```console 
    export GOOGLE_APPLICATION_CREDENTIALS="SA_KEY_PATH"
    #Run python code using Google Cloud python libraries
    python myapp.py
```

*Note*: Replace SA_KEY_PATH with the path of the JSON file that contains the service account key.


#### Compute Service assigned Service Account
If the GOOGLE_APPLICATION_CREDENTIALS is not set, the client libraries try to use the 
[assigned service account](https://cloud.google.com/docs/authentication/application-default-credentials#attached-saservice)
of the service running the code.  

This use case is meant for applications running in Google Cloud Compute solutions:  
* [Google Compute Engine](https://cloud.google.com/compute/docs/access/create-enable-service-accounts-for-instances) 
* [Kubernetes](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)
* [Cloud Run](https://cloud.google.com/run/docs/securing/service-identity)
* [Cloud Functions](https://cloud.google.com/functions/docs/securing/function-identity)  
