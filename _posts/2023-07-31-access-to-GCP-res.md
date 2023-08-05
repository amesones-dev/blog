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
 

```pyhton
# my_app.py
def explicit():
    # Import BigQuert client library
    from google.cloud import bigquery
    
    # Explicitly use service account credentials by specifying the private key file.
    client = bigquery.Client.from_service_account_json('service_account.json')
    
```     

*Note*: More information on [Authenticating as a service account](https://cloud.google.com/docs/authentication/production#auth-cloud-explicit-python)

### Implicit authentication using GOOGLE_APPLICATION_CREDENTIALS
Alternatively, you can use  client libraries methods without any credential parameters and set the variable
[GOOGLE_APPLICATION_CREDENTIALS](https://cloud.google.com/docs/authentication/application-default-credentials) in your
environment to define how to control access to Google Cloud resources.

```pyhton
# my_app.py
def implicit_():
    # Import BigQuert client library
    from google.cloud import bigquery
    
    # Client Libraries use credentials according to GOOGLE_APPLICATION_CREDENTIALS 
    client = bigquery.Client()

```        

#### Using a Service Account credential file
To access Google Cloud Services from code as an SA, set the environment variable GOOGLE_APPLICATION_CREDENTIALS to the 
JSON key file path and then run your code using Google services.  

```shell 
    export GOOGLE_APPLICATION_CREDENTIALS="SA_KEY_PATH"
    # Run python code using Google Cloud python libraries
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

    


### Example
Following the workflow in the previous post  [GCP python quickstart](2023-07-27-gcp-python-quickstart.md), suppose that 
you or your team are developing  an application that will require access to certain Google Cloud services and resources 
in a GCP project that you have previously created or have permissions to manage.

#### How to assign an identity to your application to access GCP resources 

1. [Create a service account](https://cloud.google.com/iam/docs/samples/iam-create-service-account) in the project.
   This account will be the identity used to access [Google Cloud services and resources](https://cloud.google.com/products).  
  
   For example, suppose your app will use these GCP services: 
     * [BigQuery](https://cloud.google.com/bigquery) 
     * [Datastore](https://cloud.google.com/datastore)  / [Firestore](https://cloud.google.com/firestore)

2. During the service account creation, assign the appropriate IAM  roles that will allow the 
right level of access to Google services and resources. 
    Following the example, you would grant roles to the service account according to the application requirements:
   **IAM Roles application requirements**
   * **BigQuery User**: When applied to a project, access to run queries, create datasets, read dataset metadata, and list tables. When applied to a dataset, access to read dataset metadata and list tables within the dataset.
   * **DataStore User**: Provides read/write access to data in a Cloud Datastore/Firestore database. Intended for application developers and service accounts.
   
    About [BigQuery service IAM roles](https://cloud.google.com/bigquery/docs/access-control#bigquery).
    About [Datastore service IAM roles](https://cloud.google.com/datastore/docs/access/iam#iam_roles).

3. If at some point in time your application needs access to other services or broader access, you can always add extra 
roles for the same or different services.

4. General case 
   * [Create a Service Account Key](https://cloud.google.com/iam/docs/creating-managing-service-account-keys#console) 
      for the service account using the Google Cloud console. 
      A JSON file is downloaded to your computer. This JSON file represents the private part of the key, and it will be
      used as credentials to access Google services from your development environment code. 
   * Use this Service Account key file to configure your development environment:      
      * [Explicit authentication](#explicit-authentication-from-code-with-iam-service-account-key)  
      * [Implicit authentication](#implicit-authentication-using-google_application_credentials)

5. If you are running your application in [Google Cloud Compute solutions](https://cloud.google.com/products/compute), 
consider assigning the needed roles to the service account configured for the service running the application.
   * [Compute Service assigned Service Account](#compute-service-assigned-service-account)
