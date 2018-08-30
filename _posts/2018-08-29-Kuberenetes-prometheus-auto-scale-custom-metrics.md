---
layout: post
title: Kubernetes prometheus adapter to scale based on custom metrics
tags: [Kubernetes, prometheus]
category: [Kubernetes]
author: vikrant
comments: true
--- 

In case of kubernetes we can scale the application based on CPU and memory utilization but sometimes this is not sufficient. We need other parameters specifically related to application for scaling the number of PODs. Fortunately, kubernetes prometheus has another project adapter which can be used to expose the metrics from prometheus which then can be use for scaling the app.

Here is the official repo of prometheus adapter project:  

https://github.com/DirectXMan12/k8s-prometheus-adapter 

Clone the official repo.

~~~
git clone https://github.com/DirectXMan12/k8s-prometheus-adapter.git
~~~

Since I have deployed prometheus using operator which by default have used monitoring namespace hence I need to modify the file `custom-metrics-apiserver-deployment.yaml` for correcting the `--prometheus-url` as per my requirement. 


~~~
        args:
        - --secure-port=6443
        - --tls-cert-file=/var/run/serving-cert/serving.crt
        - --tls-private-key-file=/var/run/serving-cert/serving.key
        - --logtostderr=true
        - --prometheus-url=http://prometheus-k8s.monitoring.svc:9090/
        - --metrics-relist-interval=1m
        - --v=10
        - --config=/etc/adapter/config.yaml
~~~        


Also added the following custom metric definition in `custom-metrics-config-map.yaml` so that app can scale based on total of http_requests

~~~
- seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
  resources:
    template: <<.Resource>>
  name:
    matches: http_requests_total
  metricsQuery: 'sum(rate(http_requests_total{namespace="default",pod="example-app-548896bc-stnx4"}[2m])) by (pod)'
~~~  

Here is the list of services which are present in monitoring namespace. We need the prometheus service hence the DNS name for access will become `prometheus-k8s.monitoring.svc:9090`

~~~
kubectl get svc -n monitoring
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
alertmanager-main       NodePort    10.98.23.202    <none>        9093:30900/TCP      38m
alertmanager-operated   ClusterIP   None            <none>        9093/TCP,6783/TCP   38m
grafana                 ClusterIP   10.107.76.245   <none>        3000/TCP            38m
kube-state-metrics      ClusterIP   None            <none>        8443/TCP,9443/TCP   38m
node-exporter           ClusterIP   None            <none>        9100/TCP            38m
prometheus-k8s          ClusterIP   10.100.38.122   <none>        9090/TCP            38m
prometheus-operated     ClusterIP   None            <none>        9090/TCP            38m
prometheus-operator     ClusterIP   None            <none>        8080/TCP            38m
~~~

Once you done with modification of `custom-metrics-apiserver-deployment.yaml` as per your requirement, create the resources.

~~~
$ kubectl create -f k8s-prometheus-adapter/deploy/manifests/
~~~

After that issue the following command to list all the metrics as indicated in github readme. 

~~~
$ kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/" | jq .
~~~

For testing the horizontal pod autoscaling using custom metrics clone the following repo.

~~~
git clone https://github.com/stefanprodan/k8s-prom-hpa
~~~

And use the podinfo app provided in this repo. 

https://github.com/stefanprodan/k8s-prom-hpa/tree/master/podinfo



