---
layout: post
title:  "GCP python quickstart"
date:   2023-07-27
categories: jekyll update
---
Audience: 
* You want to start using CI/CD procedures in Google Cloud Platform. 
* You are familiar with git, python, gcloud.
* You want to start coding *without any costs* for now. 

 
# GCP python quickstart guide. 
## How to create a python development environment for Google Cloud services.  
In this guide, you'll set up a Python development environment to access Google Cloud services. 
It focuses on minimal requirements, simple Google Cloud authentication and using only services that do not require 
billing enabled on the project.

## Create Google Cloud resources

1. Create a [Google Cloud](https://console.cloud.google.com/home/dashboard)  platform account if you do not already have it.

2. [Create a Google Cloud project](https://developers.google.com/workspace/guides/create-project) or use an existing one.



## Use Google Cloud Shell
To start coding right away, launch [Google Cloud Shell](https://console.cloud.google.com/home/)
Google Cloud Shell is an interactive shell environment for Google Cloud accessible from the Google Cloud Console. It 
includes the Cloud SDK, an interactive [Code Editor](https://ide.cloud.google.com), python and other Google Cloud and 
general development tools. 
 

## Use your own development environment
If you would rather use **your own local development machine**:

### Install Google Cloud SDK
[Install Google Cloud SDK](https://cloud.google.com/sdk/docs/quickstart)

### Install Python

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

## Now that your environment is set...
Whether you are using Cloud Shell or working on your local **your own local development machine**:
### Use [git](https://git-scm.com/) and [GitHub](https://github.com/) for your development environment
#### Create a git repo on Github

1. Sign in or sign up to [GitHub](https://github.com/login)
2. [Set your GitHub email address and email privacy policy](https://github.com/settings/emails).
3. [Generate personal access token](https://github.com/settings/tokens/new) to be able to push code from terminal
[Personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) are used instead of a password for Git over HTTPS or to authenticate to the git API over Basic Authentication.
Scope: repo, gist
4. Create a [new repo](https://github.com/new). 

    *Recommended for Python projects*: when creating the new repo use the option [Add .gitnore](https://docs.github.com/en/get-started/getting-started-with-git/ignoring-files) with the Python template

#### Configure git to push code to GitHub
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


### Create a pyhon virtual environment

User your cloned git repository folder for your source code and Python [venv](https://docs.python.org/3/library/venv.html)
virtual environment to isolate python dependencies. 

```
cd my-dev-project
python -m venv [venv-name]
source [venv-name]/bin/activate
```

Usual values for [venv-name] are `venv`, `dvenv`, `venv39` for a python 3.9 version virtual environment, etc.

*Note*: This folder should be excluded from git cloning, by using a wildcard in the 
[.gitnore](https://docs.github.com/en/get-started/getting-started-with-git/ignoring-files) file. 

At this point, the environment is ready to **start coding**.
```
