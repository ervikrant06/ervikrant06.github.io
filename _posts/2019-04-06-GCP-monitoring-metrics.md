---
layout: post
title: Understanding GCP (Google Cloud) monitoring
tags: [GCP]
category: [GCP]
author: vikrant
comments: true
--- 

Recently I started working on GCP (Google cloud). As we started running our workloads in GCP, monitoring becomes crucial for us. We already have on-premises solutions for monitoring. But we thought of exploring the Google stackdriver monitoring solution. When I started reading the documentation I came across various terms which can be confusing for beginner. In this article, I am demistifying those terms. Also, I am providing some helpful tips:

#### Existing monitoring options: 

- Google is already doing the monitoring of instances (VMs) running in cloud, I categorized it as a hypervisor level monitoring which is used to monitor CPU, disk and Network stats of VM. 
- Collectd agent can be installed in VM to do the extended monitoring. If you are already familiar with collectd, by-default it includes various plugins which we can enable for monitoring. 

#### Challenge which we trying to solve:

- Monitoring of the application running inside the VMs without statsd. 

#### GCP monitoring terms:

- Metric:

A metric is a measurement of some characteristic of your cloud application. For example, Monitoring has metrics for the CPU utilization of your VM instances, the number of tables in your SQL databases, and hundreds more. A metric's measurement is not a single value, because metric values come from many sources and at many times. For example, the CPU utilization metric has measurements from different VM instances, taken at different times. To tackle this multi source challenge various options are available like metric type, resource type. 

Following snippet shows an example of CPU utilization metrics from one instance. Main things to note from below output for this article are `type` in resource and metrics section. We will refer to it as resource_type and metric_type. 

~~~
metric {
  labels {
    key: "instance_name"
    value: "vikrant-submit"
  }
  type: "compute.googleapis.com/instance/cpu/utilization"
}
resource {
  type: "gce_instance"
  labels {
    key: "instance_id"
    value: "8462409971416774597"
  }
  labels {
    key: "project_id"
    value: "test-qa"
  }
  labels {
    key: "zone"
    value: "us-central1-a"
  }
}
metric_kind: GAUGE
value_type: DOUBLE
points {
  interval {
    start_time {
      seconds: 1554554520
    }
    end_time {
      seconds: 1554554520
    }
  }
  value {
    double_value: 0.206945653874
  }
}
points {
  interval {
    start_time {
      seconds: 1554554460
    }
    end_time {
      seconds: 1554554460
    }
  }
  value {
    double_value: 0.206906764665
  }
}
points {
  interval {
    start_time {
      seconds: 1554554400
    }
    end_time {
      seconds: 1554554400
    }
  }
  value {
    double_value: 0.20730459172
  }
}
~~~

- Metric Types: 

GCP provides number of metric types. Let's what exactly it's with an example: Like mentioned above if we need to get the CPU information from all instances we need to have some sort of definition what we want to capture from all instances. Following one showing the kind of value which we are expecing from metrics. This metric type is associated with above metric. 

~~~
name: "projects/test-qa/metricDescriptors/compute.googleapis.com/instance/cpu/utilization"
labels {
  key: "instance_name"
  description: "The name of the VM instance."
}
metric_kind: GAUGE
value_type: DOUBLE
unit: "1"
description: "The fraction of the allocated CPU that is currently in use on the instance. This value can be greater than 1.0 on some machine types that allow bursting."
display_name: "CPU utilization"
type: "compute.googleapis.com/instance/cpu/utilization"
metadata {
  launch_stage: GA
  sample_period {
    seconds: 60
  }
  ingest_delay {
    seconds: 240
  }
}
~~~

- Resource Type:

Resource type is used to add the resource information in metric captured from a resource. If you see the metric snippet shown above, it get all the project_id, instance_id and zone information from the RT (resource type) defintion. 

~~~
type: "gce_instance"
display_name: "GCE VM Instance"
description: "A virtual machine instance hosted in Google Compute Engine (GCE)."
labels {
  key: "project_id"
  description: "The identifier of the GCP project associated with this resource, such as \"my-project\"."
}
labels {
  key: "instance_id"
  description: "The numeric VM instance identifier assigned by Compute Engine."
}
labels {
  key: "zone"
  description: "The Compute Engine zone in which the VM is running."
}
name: "projects/test-qa/monitoredResourceDescriptors/gce_instance"
~~~

#### Coming back to problem statement

- Since none of the existing metric type was available for application monitoring hence we created the custom metric type. 

~~~
#!/usr/bin/python
from google.cloud import monitoring_v3
def create_negotiator_metric():
    client = monitoring_v3.MetricServiceClient()
    project_name = client.project_path(PROJECT_ID)
    condor_negotiator = monitoring_v3.types.MetricDescriptor()
    condor_negotiator.type = 'custom.googleapis.com/negotiator_cycle'
    condor_negotiator.metric_kind = monitoring_v3.enums.MetricDescriptor.MetricKind.GAUGE
    condor_negotiator.value_type = monitoring_v3.enums.MetricDescriptor.ValueType.INT64
    condor_negotiator.description = "Duration of negotiator cycle"
    condor_negotiator = client.create_metric_descriptor(project_name, condor_negotiator)
    print('Created {}.'.format(condor_negotiator.name))
create_negotiator_metric()
~~~

Checking the definition of our customer metric type:

Code: 

~~~
from google.cloud import monitoring_v3
client = monitoring_v3.MetricServiceClient()
project_id=PROJECT_ID
project_name = client.project_path(PROJECT_ID)
descriptor_negotiator=client.get_metric_descriptor('projects/PROJECT_ID/metricDescriptors/custom.googleapis.com/negotiator_cycle')
print(descriptor_negotiator)
~~~

Output:

~~~
name: "projects/trc-hpc-qa/metricDescriptors/custom.googleapis.com/negotiator_cycle"
metric_kind: GAUGE
value_type: INT64
description: "Duration of negotiator cycle"
type: "custom.googleapis.com/negotiator_cycle"
~~~

- GCP provides two generic options for resource type `generic_node` and `generic_task`. For application monitoring we used generic_task. 5 labels are available with this RT. 

Code: 

~~~
from google.cloud import monitoring_v3
client = monitoring_v3.MetricServiceClient()
project_id=PROJECT_ID
project_name = client.project_path(PROJECT_ID)
resource_path = client.monitored_resource_descriptor_path(
    project_id, 'generic_task')
print(client.get_monitored_resource_descriptor(resource_path))
~~~

Output:

~~~
type: "generic_task"
display_name: "Generic Task"
description: "A generic task identifies an application process for which no more specific resource is applicable, such as a process scheduled by a custom orchestration system. The label values must uniquely identify the task."
labels {
  key: "project_id"
  description: "The identifier of the GCP project associated with this resource, such as \"my-project\"."
}
labels {
  key: "location"
  description: "The GCP or AWS region in which data about the resource is stored. For example, \"us-east1-a\" (GCP) or \"aws:us-east-1a\" (AWS)."
}
labels {
  key: "namespace"
  description: "A namespace identifier, such as a cluster name."
}
labels {
  key: "job"
  description: "An identifier for a grouping of related tasks, such as the name of a microservice or distributed batch job."
}
labels {
  key: "task_id"
  description: "A unique identifier for the task within the namespace and job, such as a replica index identifying the task within the job."
}
name: "projects/test-qa/monitoredResourceDescriptors/generic_task"
~~~

- Time to put the metrics from application log file into custom metrics. 

~~~
import subprocess
import time
from google.cloud import monitoring_v3

def negotiator_cycle_duration(duration):
    client = monitoring_v3.MetricServiceClient()
    series = monitoring_v3.types.TimeSeries()
    series.metric.type = 'custom.googleapis.com/negotiator_cycle'
    series.resource.type = 'generic_task'
    series.resource.labels['location'] = 'us-central1-a'
    series.resource.labels['namespace'] = 'test1'
    series.resource.labels['job'] = 'vikrant-cluster'
    series.resource.labels['task_id'] = "negotiator_cycle"
    point = series.points.add()
    point.value.int64_value = duration
    now = time.time()
    point.interval.end_time.seconds = int(now)
    point.interval.end_time.nanos = int((now - point.interval.end_time.seconds) * 10**9)
    client.create_time_series('projects/<PROJECTID>/metricDescriptors/', [series])
def negotiator_duration():
    cmd = "grep 'Phase 4.1' /apps/test/vaggarwal/Downloads/negotiator.log | tail -2 | awk '{{print $1,$2}}'"
    ps = subprocess.Popen(cmd, shell=True, stdout=subprocess.PIPE, stderr=subprocess.STDOUT)
    output = ps.communicate()[0]
    t1, t2 = map(lambda x: time.mktime(time.strptime(x, "%x %X")), output.strip().split("\n"))
    cycle_duration = int(t2 - t1)
    negotiator_cycle_duration(cycle_duration)
negotiator_duration()
~~~

Lets see the metric which our script pushed to GCP:

Code: 

~~~
#!/usr/bin/python
import time
from google.cloud import monitoring_v3
client = monitoring_v3.MetricServiceClient()
project_name = client.project_path(PROJECT_ID)
interval = monitoring_v3.types.TimeInterval()
now = time.time()
interval.end_time.seconds = int(now)
interval.end_time.nanos = int(
    (now - interval.end_time.seconds) * 10**9)
interval.start_time.seconds = int(now - 17200)
interval.start_time.nanos = interval.end_time.nanos
results = client.list_time_series(
    project_name,
    'metric.type = "custom.googleapis.com/negotiator_cycle"',
    interval,
    monitoring_v3.enums.ListTimeSeriesRequest.TimeSeriesView.FULL)
for result in results:
    print(result)
~~~

Output: 

~~~
metric {
  type: "custom.googleapis.com/negotiator_cycle"
}
resource {
  type: "generic_task"
  labels {
    key: "job"
    value: "vikrant-cluster"
  }
  labels {
    key: "location"
    value: "us-central1-a"
  }
  labels {
    key: "namespace"
    value: "test1"
  }
  labels {
    key: "project_id"
    value: "test-qa"
  }
  labels {
    key: "task_id"
    value: "negotiator_cycle"
  }
}
metric_kind: GAUGE
value_type: INT64
points {
  interval {
    start_time {
      seconds: 1554548336
      nanos: 356986000
    }
    end_time {
      seconds: 1554548336
      nanos: 356986000
    }
  }
  value {
    int64_value: 20
  }
}
~~~