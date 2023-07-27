---
layout: post
title:  "GCP python quickstart"
date:   2023-07-27
categories: jekyll update
---
# GCP python quickstart guide. 
## How to create a python development environment for Google Cloud services.  


In this guide, you'll set up a local Python development to access Google Cloud services. It focuses on minimal requirements, 
simple Google Cloud authentication and using only services that do not require billing enabled on the project.


## Create Google Cloud resources

1. Create a [Google Cloud](https://console.cloud.google.com/home/dashboard)  platform account if you do not already have it.

2. [Create a Google Cloud project](https://developers.google.com/workspace/guides/create-project) or use an existing one.

3. [Create a service account](https://cloud.google.com/iam/docs/samples/iam-create-service-account) in the project just created .

   This account will be the identity used to access Google Cloud Services.
   
   For example: 
     * [BigQuery](https://cloud.google.com/bigquery) 
     * [Datastore](https://cloud.google.com/datastore)  / [Firestore](https://cloud.google.com/firestore)

    During the service account creation, assign the following roles that will allow the 
    right level of access to Google services:
	
    Roles
    * **BigQuery User**: When applied to a project, access to run queries, create datasets, read dataset metadata, and list tables. When applied to a dataset, access to read dataset metadata and list tables within the dataset.
    * **DataStore User**: Provides read/write access to data in a Cloud Datastore/Firestore database. Intended for application developers and service accounts.

5. [Create a Service Account Key](https://cloud.google.com/iam/docs/creating-managing-service-account-keys#console)  for the service account using the Google Cloud console. 

   A JSON file is downloaded to your computer. This JSON file represents the private part of the key, and it will be used as credentials to access Google services from your development environment code.
For now, make a note of the file location as it will be used later on in the example code.

    *Note*: More information on [Authenticating as a service account](https://cloud.google.com/docs/authentication/production#auth-cloud-explicit-python)


## Install Google Cloud SDK
**On your local development machine**

[Install Google Cloud SDK](https://cloud.google.com/sdk/docs/quickstart)


## Install Python

*Note*: Console snippets for Debian/Ubuntu based distributions.

**On your local development machine**

1. Install python packages.

    ```console
    sudo apt update
    sudo apt install python3 python3-dev python3-venv
    ```
    
2. Install pip 

    *Note*: Debian provides a package for pip

    ```console
    sudo apt install python-pip
    ```
    Alternatively pip can be installed with the following method
    ```console
    wget https://bootstrap.pypa.io/get-pip.py
    sudo python3 get-pip.py
    ```

## Use [git](https://git-scm.com/) and [GitHub](https://github.com/) for your development environment
### Create a git repo on Github

1. Sign in or sign up to [GitHub](https://github.com/login)
2. [Set your GitHub email address and email privacy policy](https://github.com/settings/emails).
3. [Generate personal access token](https://github.com/settings/tokens/new) to be able to push code from terminal
[Personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) are used instead of a password for Git over HTTPS or to authenticate to the git API over Basic Authentication.
Scope: repo, gist
4. Create a [new repo](https://github.com/new). 

    *Recommended for Python projects*: when creating the new repo use the option [Add .gitnore](https://docs.github.com/en/get-started/getting-started-with-git/ignoring-files) with the Python template

### Configure git to push code to GitHub
**On your local development machine**
1. [Set your commit email address](https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-github-user-account/managing-email-preferences/setting-your-commit-email-address)

    ```console
    git config --global user.name "yourname"
    git config --global user.email "email@example.com"    
    ```


3. Use personal access token and git primary email address to authenticate git operations.
    ```console
    git config --global user.email "email@example.com"  
    git clone https://github.com/[git-user-name]/[my-dev-project].git 
    Username: [git configured email address]
    Password: [Your personal access token]
    ```
    *Note*: If you changed your git email address or email address policy after cloning the repository use this command to update commits author
    ```console
    git commit --amend --reset-author
    ```
**You are now able to author and push code from your local machine to yout GitHub repository**.


## Create a development environment

**On your local development machine**

User your cloned git repository folder for your source code and Python [venv](https://docs.python.org/3/library/venv.html) virtual environment to isolate python dependencies. 

```
cd my-dev-project
python -m venv [venv-name]
source [venv-name]/bin/activate
```

Usual values for venv-name are `venv`, `dvenv`, `venv39` for a python 3.9 version virtual environment, etc.

*Note*: This folder should be excluded from git cloning, by using a wildcard in the [.gitnore](https://docs.github.com/en/get-started/getting-started-with-git/ignoring-files) file. 

At this point, the environment is ready to start coding.

## About authenticating to Google Cloud services
### Implicit authentication using environment variables
The [Python Cloud Client Libraries](https://cloud.google.com/python/docs/reference?l=python) are designed to use
the environment variable `GOOGLE_APPLICATION_CREDENTIALS`  to locate the Service Account Key JSON file. 
To access Google Cloud Services from code as the service account, set the environment variable to the JSON key file path
and then run your code using Google services.

```console 
    export GOOGLE_APPLICATION_CREDENTIALS="KEY_PATH"
    #Run python code using Google Cloud python libraries
    python myapp.py
```
*Note*: Replace KEY_PATH with the path of the JSON file that contains the service account key.


### Explicit authentication from code
You can alternately choose to explicitly point to your service account file in code.
Google Cloud Client libraries include methods to use a Service Account Key json file as credential when creating 
client objects. 

```code
def explicit():
    #Import BigQuert client library
    from google.cloud import bigquery
    
    # Explicitly use service account credentials by specifying the private key file.
    client = bigquery.Client.from_service_account_json('service_account.json')
    
```
