---
layout: post
title: How to write python code GCP (Google cloud)
tags: [GCP]
category: [GCP]
author: vikrant
comments: true
--- 

In [previous post](https://ervikrant06.github.io/gcp/GCP-monitoring-metrics/), I shared some information about the stackdriver monitoring metrics. In this post, I am providing a tip of automating the tasks in GCP. Recently I was assigned a taks of automating the creation of GCP log sink, dataset creation and exclusion filter creation. I started looking around the documentation and was initially confused about the best method of achieving it. I saw python examples in official guide and tentalized to use it for doing my job.

A snippet from my script : 

~~~
#!/usr/bin/python

import json
import os
import requests
from google.cloud import bigquery, logging
from googleapiclient import discovery
from oauth2client.client import GoogleCredentials

project_id=12345
project_name="test-qa"
credpath="/apps/ttech/vaggarwal/creds/test-qa.json"


filter="""resource.type="gce_instance" AND
          logName=("projects/test-qa/logs/condor_negotiatorlog" OR
          "projects/test-qa/logs/condor_scheddrestartreport" OR
          "projects/test-qa/logs/condor_schedlog" OR
          "projects/test-qa/logs/condor_shadowlog" OR
          "projects/test-qa/logs/condor_xferstatslog" OR
          "projects/test-qa/logs/syslog" OR
          "projects/test-qa/logs/salt_logs")"""

def set_creds(credpath):
    """
    Summary line:
    Set the credential file path if it's not through env variable.
    Parameters:
    credpath : location of credential file.
    Returns:
    Nothing
    """
    if os.environ.get('GOOGLE_APPLICATION_CREDENTIALS') is None:
        os.environ['GOOGLE_APPLICATION_CREDENTIALS'] = credpath

def create_dataset(datasetname):
    """
    Summary line:
    Verify dataset is not already present and then create dataset.
    Parameters:
    dataset_id : Name of the dataset
    Returns:
    output_ref_object : dataset reference
    """
    client = bigquery.Client()
    dataset_ref = client.dataset(datasetname)
    datasets = list(client.list_datasets())
    for each_dataset in datasets:
        if each_dataset.dataset_id == datasetname:
            print("Dataset is already present")
            return
    dataset = bigquery.Dataset(dataset_ref)
    dataset.location = "US"
    dataset.description = "test dataset"
    dataset.default_table_expiration_ms = 604800000
    dataset = client.create_dataset(dataset)
    newdataset_ref_object = client.get_dataset(dataset)
    return(newdataset_ref_object)
~~~

This was not providing all the functionalities which I was looking for:

- Like allowing the creation of dataset in another project.
- Assiging sink service account user role in datset.
- Not able to create exclusion. 
- Was bit complex to deal with output.

I started exploring further and one of my colleague told me about this magically link:

https://developers.google.com/apis-explorer/#p/

You just need to search for any component in it like logging, compute and you use the respective list, create API depending upon your you case. Once you identified the body of your API request then it's very easy to write the code. For non-programmer like me, it took only 30-40 mins to come with following code. I would suggest to explore the API in above link and then try to understand the code.

~~~
#!/usr/bin/python

import json
import os
import requests
from google.cloud import bigquery, logging
from googleapiclient import discovery
from oauth2client.client import GoogleCredentials

project_id=123456
project_name="test-qa"
credpath="/apps/ttech/vaggarwal/creds/test-qa.json"


filter="""resource.type="gce_instance" AND 
          logName=("projects/test-qa/logs/condor_negotiatorlog" OR 
          "projects/test-qa/logs/condor_scheddrestartreport" OR 
          "projects/test-qa/logs/condor_schedlog" OR 
          "projects/test-qa/logs/condor_shadowlog" OR 
          "projects/test-qa/logs/condor_xferstatslog" OR 
          "projects/test-qa/logs/syslog" OR 
          "projects/test-qa/logs/salt_logs")"""
          
def set_creds(credpath):
    if os.environ.get('GOOGLE_APPLICATION_CREDENTIALS') is None:
        os.environ['GOOGLE_APPLICATION_CREDENTIALS'] = credpath

def create_sink(dataset_name, sink_name, filter):
    credentials = GoogleCredentials.get_application_default()
    service=discovery.build('logging','v2', credentials=credentials)
    project_path="projects/" + project_name
    body={
        "filter": filter,
        "name": sink_name,
        "destination": "bigquery.googleapis.com/projects/{}/datasets/{}".format(project_name, dataset_name)
    }
    output=service.projects().sinks().create(body=body,parent=project_path).execute()
    stack_service_account=output['writerIdentity']
    print(stack_service_account)
    return(stack_service_account.split(':')[1])
    
def create_dataset(dataset_name, stack_service_account):
    credentials = GoogleCredentials.get_application_default()
    service=discovery.build('bigquery','v2', credentials=credentials)
    body={
        "datasetReference": {
            "datasetId": dataset_name,
            "projectId": project_name
        },
        "defaultTableExpirationMs": "604800000",
        "location": "US",
        "access": [
            {
            "role": "roles/bigquery.dataEditor",
            "userByEmail": stack_service_account
            }
        ]
    }
    service.datasets().insert(body=body,projectId=project_name).execute()

def create_exclusion(exclusion_name, filter):
    credentials = GoogleCredentials.get_application_default()
    body={
        "name": exclusion_name, 
        "filter": filter, 
    }
    project_path="projects/" + project_name
    service=discovery.build('logging','v2', credentials=credentials)
    service.exclusions().create(body=body,parent=project_path).execute()


if __name__ == '__main__':
    
    team_name=project_name.split('-')[-1]
    dataset_name="trc_hpc_3_" + team_name
    sink_name=dataset_name
    exclusion_name=dataset_name
    stack_service_account=create_sink(dataset_name, sink_name, filter)
    print(stack_service_account)
    create_dataset(dataset_name, stack_service_account)
    create_exclusion(exclusion_name, filter)
~~~