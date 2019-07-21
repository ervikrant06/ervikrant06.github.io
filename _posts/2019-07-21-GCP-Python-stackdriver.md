---
layout: post
title: How to use stackdriver logging and error reporting in python?
tags: [GCP]
category: [GCP, Python]
author: vikrant
comments: true
--- 

Stackdriver is another wonderful product from google. We are using stackdriver for centralized logging from all VMs and bq for analytics on the collected logs. We are running bunch of scripts to dynamically start the instances depending upon the workload, we recently started exploring using stackdriver as a logging handler in the scripts to send all the logs to stackdriver. Stackdriver is not only about logging and monitoring; other hidden useful bits of stackdriver are error reporting, profiling, tracing and debugger. Last three features are used for application monitoring. In this article I am sharing information on the usage of stackdriver logging and error reporting. 


I prepared simple python script for demostration of the usage of logging and error reporting. I have used cloud shell to prepare this script. Script is used to create the bucket if the bucket is created successfully then it will send a message to stackdriver in "GCE VM instances" resource type apply a filter of 'BucketAccessLog' and if it fails then it will send a call trace to stackdriver error reporting. 


```
#!/usr/bin/python
import sys
import google.cloud.logging
from google.cloud import storage
from google.cloud import error_reporting
from google.cloud.logging.resource import Resource


storage_client = storage.Client()
error_reporting_client = error_reporting.Client()
logging_client = google.cloud.logging.Client()

bucket_name = sys.argv[1]

dict1 = {
        "message": "Null"
        }
res = Resource(
        type="gce_instance",
        labels={
            "location": "us-central-1"
            },
        )

logger = logging_client.logger('BucketAccessLog')
def log_message(dict2):
    logger.log_struct(
        dict2, resource=res
    )

try:
    bucket_details = storage_client.create_bucket(bucket_name)
    message = "Bucket created successfuly {}".format(bucket_details.name)
    dict1['message']=message
    log_message(dict1)
except Exception:
    error_reporting_client.report_exception()
```    