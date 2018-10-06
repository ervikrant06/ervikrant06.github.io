---
layout: post
title: How to add K8 consul client to external consul cluster
tags: [consul, kubernetes]
category: [consul, kubernetes]
author: vikrant
comments: true
--- 

This article is bit different than the previous articles, in this case we have deployed the consul cluster on docker-machine, you may refer [this blog post](https://ervikrant06.github.io/consul/Consul/) for it. 

- After that do the following modifications in values.yaml file. IMP change is specifying the ip address of external consul cluster and disabling the consul server setup on K8. 

~~~
diff --git a/values.yaml b/values.yaml
index 55e6dfe..a2fcac4 100644
--- a/values.yaml
+++ b/values.yaml
@@ -9,7 +9,7 @@ global:
   # will enable or disable all the components within this chart by default.
   # Each component can be overridden using the component-specific "enabled"
   # value.
-  enabled: true
+  enabled: false

   # Domain to register the Consul DNS server to listen for.
   domain: consul
@@ -22,7 +22,7 @@ global:
   # as. This shouldn't be changed once the Consul cluster is up and running
   # since Consul doesn't support an automatic way to change this value
   # currently: https://github.com/hashicorp/consul/issues/1858
-  datacenter: dc1
+  datacenter: labsetup

 server:
   enabled: "-"
@@ -72,9 +72,9 @@ server:
 # within the Kube cluster. The current deployment model follows a traditional
 # DC where a single agent is deployed per node.
 client:
-  enabled: "-"
+  enabled: true
   image: null
-  join: null
+  join: ["192.168.99.101"]

   # Resource requests, limits, etc. for the client cluster placement. This
   # should map directly to the value of the resources field for a PodSpec.
@@ -103,6 +103,9 @@ client:
 dns:
   enabled: "-"

+syncCatalog:
+  enabled: true
+
 ui:
   # True if you want to enable the Consul UI. The UI will run only
   # on the server nodes. This makes UI access via the service below (if
~~~

- Deploy the helm chart. 

~~~
helm install --name consulclient2 .
NAME:   consulclient2
LAST DEPLOYED: Sat Oct  6 16:48:59 2018
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Pod(related)
NAME                 READY  STATUS             RESTARTS  AGE
consulclient2-kfn6z  0/1    ContainerCreating  0         1s

==> v1/ConfigMap
NAME                         DATA  AGE
consulclient2-client-config  1     1s

==> v1/DaemonSet
NAME           DESIRED  CURRENT  READY  UP-TO-DATE  AVAILABLE  NODE SELECTOR  AGE
consulclient2  1        1        0      1           0          <none>         1s
~~~

- Check the membership of external cluster. It's showing K8 consul client as a part of consul cluster. 

~~~
docker@consul1:~$ docker exec -it 2b3648bc46b0 consul members -http-addr=192.168.99.101:8500
Node                 Address              Status  Type    Build  Protocol  DC        Segment
consul1              192.168.99.101:8301  alive   server  1.2.3  2         labsetup  <all>
consulclient2-kfn6z  172.17.0.10:8301     alive   client  1.2.2  2         labsetup  <default>
~~~