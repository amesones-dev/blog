---
layout: post
title:  "Publishing BigQuery dataset with FastAPI"
date:   2023-23-08
categories: jekyll update
---

Audience:
* You want to create professional APIs
* You want to leverage datasets from BigQuery
* You are familiar with BigQuery, sql, python, FastAPI
* You want to start coding *without any costs* for now.

In this guide, we will 
* showcase how to provide content to an application or an API using  Google Cloud services, in this case, BigQuery
* and at the same time explore concepts related to professional API development


## BigQuery
[BigQuery](https://cloud.google.com/bigquery) is a Google Cloud data warehouse system, perfect for Big Data analysis. BigQuery  
* allows creating and maintaining large datasets and running SQL queries to extract data insights
* hosts public datasets of projects in many areas of research. 
* offers a free tier service called [BigQuery sandbox](https://cloud.google.com/bigquery/docs/sandbox) for hosting
and querying your own datasets.
* ...and much [more](https://cloud.google.com/bigquery/docs/introduction) 


## FastAPI
FastAPI is a python framework for building professional APIs quickly.

FastAPI:
* Produces modern API standards compliant code, like [OpenAPI](https://github.com/OAI/OpenAPI-Specification)
* is [modular](https://fastapi.tiangolo.com/tutorial/bigger-applications/), making new endpoints and API features seamless.
* is [JSON compatible](https://fastapi.tiangolo.com/tutorial/encoder/)
* has a suitable and progressive leaning curve, and [excellent learning resources](https://fastapi.tiangolo.com/tutorial/)
* is compatible with [popular python frameworks](https://fastapi.tiangolo.com/advanced/wsgi/)


## The FastAPI example
The example, [gfs-bq-fastapi](https://github.com/amesones-dev/gfs-bq-fastapi.git),  is a minimal API that shows how to 
publish BigQuery dataset content with FastAPI.  

* It shows how to connect to Google Cloud BigQuery seamlessly from an application, a FastAPI app in this case, 
using a BigQuery connections manager class.
 
* It runs a simple FastAPI illustrating foundational features like:
  * using FastAPI modules
  * using JSON payload encoding

### GBQManager class

**GBQManager**
1. Creates a  BigQuery API client from a service account key file  
  *Note: required IAM roles for service account: BigQuery User*
2. Isolates BigQuery connections management

**Class use example to manage BigQuery connections for an app**  

*Link GBQManager to app*
```python
    # App specific
    # Link GBQManager to app
    bq = GBQManager()
    bq.init_app(self.app)
```

*Use GBQManager to run BigQuery jobs*    
```python    
    sql_query = """SELECT DISTINCT country_region  
                FROM `bigquery-public-data.covid19_jhu_csse_eu.summary`  
                ORDER BY country_region ASC""" 
    query_job = bq.client.query(query=sql_query)
```

### AppBQContentManager class
**AppBQContentManager**  

Basic content manager that isolates content metadata management and provides basic content cache.
1. Maintains a list of contents, content titles and SQL queries definitions used to generate them.
2. Uses a GBQManager to manage BigQuery connections.
 
**Class use example to publish BigQuery data**  

*Link BigQueryManager to app*
```python
    # Basic Big Query sourced data in memory content manager
    # Loads data from BQ using a preconfigured set of titles and SQL queries
    # Basic management of content freshness to avoid duplicated running BigQuery sql queries
    # Possible improvement: adding content cache service (redis, memcache)
    
    from gbq_manager import GBQManager
    from fastapi.encoders import jsonable_encoder
    ...

    app_bq_cm = AppBQContentManager()
    app_bq_cm.init_app(app=fastapi_app, bq=GBQManager())

    # key identifies uniquely a content (title, SQL query to generate content data)
    # the load_content function uses GBQManager to run the BigQuery SQL queries
    payload = app_bq_cm.load_content(key=key, country=country)

```
### FastAPI and BigQuery integration
```python
def create_app(configclass=Config) -> FastAPI:
    root_app = FastAPI()
    root_app.config = configclass().to_dict()

    # Create a BigQuery connection manager and link to this app
    bq = GBQManager()
    bq.init_app(root_app)

    # Big Query data  content manager
    app_bq_cm = AppBQContentManager()
    app_bq_cm.init_app(root_app, bq=bq)
```

### FastAPI modular configuration
FastAPI allows extending API features by adding modules, called 
[APIRouters](https://fastapi.tiangolo.com/tutorial/bigger-applications/#apirouter), that deal with API operations for 
specific set of items. With API routers you can add endpoints to manage operations on the specific set of items and their
backing, if any, data store support.

*API Routers use case*  
Suppose that you are managing an API and there's a new business requirement to manage a different set of items, 
for example, users subscriptions with a different database to the original set of items. 
You can write an API Router for the subscriptions with new endpoints and later integrate it in the original FastAPI,
reusing the original general API configuration and settings. 

*Root endpoints*
```python
    # Fast API config
    @root_app.get("/")
    ...
    @root_app.get("/healthcheck")
    ...
    
    # Example of using modular routes with router
    root_app.include_router(countries.router)
```
*Module definition*
```python
    # countries.py
    from fastapi import APIRouter
    ...
    router = APIRouter()
    
    @router.get("/countries/", tags=["countries"])
    def get_countries():
        return countries()
```

### FastAPI: isolating data layer and JSON encoding

```python
def countries():
    ...
    # bq_cm is the application BigQuery content manager
    # key identifies specific content in the content manager store
    payload = bq_cm.load_content(key=key)
    return jsonable_encoder(payload)
```


### Running the FastAPI example  
Follow these  [instructions](https://github.com/amesones-dev/gfs-bq-fastapi#readme) to run the application locally or 
[run it with Docker](https://github.com/amesones-dev/gfs-bq-fastapi/blob/main/run/README_DOCKER_RUN.md#instructions).


### Inspect API definition
At this point your API is published and the endpoints ready to receive requests.  
* Check the /docs endpoint to see the API definition 
* or /openapi.json to get the OpenAPI json file for your newly created API.

**Example application**
![Example application](/blog/res/img/gfsBQfastAPIdemo.png)
