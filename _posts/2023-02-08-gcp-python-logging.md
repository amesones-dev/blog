---
layout: post
title:  "Cloud logging from the start"
date:   2023-08-02
categories: jekyll update
---
Audience: 
* You want to start using CI/CD procedures in Google Cloud Platform.
* You are familiar with SRE principles
* You recognise the importance of logging and monitoring for efficient service operation
* You are familiar with gcloud, python.
* You want to start coding *without any costs* for now. 

## Using Google Cloud resources to add Cloud logging to an app 
#### Leveraging free resources to apply SRE  
In this guide you will incorporate Google Cloud logging to a simple python application. 
### SRE in mind
Adding logging and monitoring to your applications is a basic step when applying [SRE](https://sre.google/) principles to
applications development. App logging and monitoring are needed to define, create and measure [SLOs](https://cloud.google.com/stackdriver/docs/solutions/slo-monitoring#defn-slo).
In GCP, [Google Cloud Operations]( https://cloud.google.com/products/operations) provides, among other things, logging and monitoring with 
[Google Cloud Logging](https://cloud.google.com/logging) and [Google Cloud Monitoring](https://cloud.google.com/monitoring). 

A good practice is to always **add logging** and monitoring capabilities to your team applications **from the start**, 
even for small projects, pilots, proofs of concept, etc. In general, GCP APIs are ready to scalate easily, so if your 
proof of concept gets to become a product, your app Ops tooling is ready and requires minimal change to go to production.  

### Google Cloud Logging  

Although there are many commercial solutions for logging and Ops in general, GCP offers  
* A powerful Cloud Logging API 
* Out of the box tools to create Ops Dashboards
* A great variety of log data sinks to use your apps logs in analytics and data pipelines
* a nice quota of log storage

And on top of all:
* Many of these utilities are based on OpenSource projects
  * Assuring compatibility 
  * Preventing technology lock
  * Allowing multi-cloud compatibility
    * You can log to Google Cloud from any  other platform.  
      You could have applications running in [Amazon EKS](https://aws.amazon.com/eks/) and manage those
      application logs, create Ops dashboards based on them and even have a dedicated team dealing with Ops in GCP.

* You can start using all these resources for free

### Adding Google Cloud Logging to an application 

When coding, writing  logs to Google Cloud Logging services can be done  in two ways:
1. Using the Python logging handler included with the Logging client library
   * [Connecting the Google Cloud Logging library to Python logging](https://cloud.google.com/logging/docs/setup/python#connecting_the_library_to_python_logging)
   * [Using the Python root logger](https://cloud.google.com/logging/docs/setup/python#using_the_python_root_logger)
2. Using [Cloud Logging API Cloud client library](https://cloud.google.com/logging/docs/setup/python#use_the_cloud_client_library_directly) for Python directly.


#### Recommended practice for writing app logs
* Google recommends integrating Google Cloud Logging with the standard python logging library for 
 [writing app logs](https://cloud.google.com/appengine/docs/standard/python3/writing-application-logs#writing_app_logs):
  * Allows reusing code modules with standard python logging methods.
  * Once the Google Cloud Logging has been set up to use the Python root logger,no specific code is needed to log to 
  Google Cloud
* For apps whose specific purpose is related to Google Cloud Logging, such as application logs analytics or logging 
 configuration management, the [Google Cloud Logging API](https://cloud.google.com/logging/docs/reference/libraries) 
 provides advanced logs manipulation methods.


* When working on GCP, whether with standalone projects or within a GCP Organization, the best practice is to have one
or more centralized Ops projects just for logging, monitoring, dashboards, alerts and Operation in general.  
* Logging to a specific project is achieved by choosing the appropriate credentials with the Cloud Logging API, 
regardless of any other resources and projects the application is accessing in GCP.  


## Cloud logging implementation

The [gfs_log_manager](https://github.com/amesones-dev/gfs_log_manager.git) demo application uses a Google Cloud Logging handler
integrated with standard python logging and uses methods form python standard logging module by implementing a class 
called GLogManager.  
For a more detailed information about the implementation  check the repo 
[README](https://github.com/amesones-dev/gfs_log_manager#readme)  


#### GLogManager class
1. Creates a Google Cloud Logging client from a service account key file  for a SA with the IAM role Logs Writer
2. Creates a Google Cloud Log handler that writes logs to a Google Cloud log with a specific log name
3. Integrates the handler with the standard python Logging class
4. Creates a google.cloud.logging.Logger to use the Cloud Logging API directly if so desired

#### Class use example to add logging to an app  
*Link GLogManager to app*
```python
    from glog_manager import GLogManager
    # Link Google Cloud Manager to app
    gl = GLogManager()
    gl.init_app(app)
    # From now on, when using standard logging in the app code
    # Log messages are stored in a Google Cloud Project
    # Defined by the app configuration
```     
*Use GLogManager to log to GCP*    
```python    
    info_msg = self.app_log_id + ':starting up'
    warn_msg = self.app_log_id + ':warning message test'
    error_data = {"url": "http://test.example.com", "data": "Test error", "code": 403}

    # Logging to GCP using root Cloud Logging standard logging integration
    logging.info(info_msg)
    logging.warning(warn_msg)
    logging.error(error_data)
    
    # Logging to GCP using Cloud Logging API directly
    gl.cloud_logger.log_text(info_msg, severity='INFO')
    gl.cloud_logger.log_text(warn_msg, severity='WARNING')
    gl.cloud_logger.log_struct(error_data, severity='ERROR')
```
#### Cloud Logging  configuration  
The main configuration settings are:
* GCP service account key file path: the Service Account determines the GCP project where the log messages will be stored.
* GCP logger name: the specific name of the log where the log messages will be stored.


```shell
   # Google Cloud Logging service account key json file
   # Determines service account and hence GCP project where logs are stored
    LG_SA_KEY_JSON_FILE = os.environ.get('LG_SA_KEY_JSON_FILE') or '/etc/secrets/sa_key_lg.json'

    # Google Cloud Log Name configuration which must follow Google Cloud log names constraints
    # If it does not exists yet, a new log will be created in GCP project with this name
    GC_LOGGER_NAME = os.environ.get('GC_LOGGER_NAME') or 'gcpLogDemo'
```

## Cloud logging reading and management
* You can use [Google Cloud Logs Explorer](https://cloud.google.com/logging/docs/view/logs-explorer-interface) and select
your GCP project and log name configured for the application to explore the logs generated.  
 
![Application logs in GCP](/blog/res/img/log_explorer_demo.jpg)
 
* You can use the [gcloud logging](https://cloud.google.com/logging/docs/reference/tools/gcloud-logging) set of commands in [Google Cloud Shell](https://console.cloud.google.com/home/) or your
local  environment with [Google Cloud SDK](https://cloud.google.com/sdk/docs/quickstart)

```shell
export PROJECT_ID=YOUR_PROJECT_ID
# GC_LOGGER_NAME was defined in code in the app configuration
export GC_LOGGER_NAME=YOUR_LOGGER_NAME

# Reading logs
LOGFILTER="logName=projects/${PROJECT_ID}/logs/${GC_LOGGER_NAME} AND severity>=WARNING"
export MAX_LOG_ENTRIES=5
export FRESHNESS=30d

gcloud logging read --freshness=30d  --limit $MAX_LOG_ENTRIES  "$LOGFILTER" --format json
...
    {   ...
        "jsonPayload": {
          "code": 403, "data": "Test error", "url": "http://test.example.com"
        },
        "logName": "projects/YOUR_PROJECT_ID/logs/gcpLogDemo",     
        "resource": { "labels": { "project_id": "YOUR_PROJECT_ID"},},
        "severity": "ERROR",
        ...     
      },
...

```
## For a future occasion...
* How to create and manage log sinks
* How to create monitoring alerts based on log metrics
* How to manage logs in a GCP organization
 
