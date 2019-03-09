---
layout: post
title: How we are using Calico CNI plugin in K8?
tags: [Kubernetes]
category: [Kubernetes]
author: vikrant
comments: true
---

Calico is a CNI (Container network interface) plugin which can be used for K8 to provide the network capabilities to K8 PODs.  We are running approx 600 node baremetal K8 cluster in production. On each node, we have one stateful POD running and we are running many batch jobs inside the POD. Calico CNI plugin was chosen in order to provide the networking among the K8 nodes without setting up vxlan.

Following the are CRD (customer resource defintions) provided by Calico.

~~~
$ kubectl get crd -n kube-system | grep -i calico
bgpconfigurations.crd.projectcalico.org       2018-06-08T15:26:57Z
bgppeers.crd.projectcalico.org                2018-06-08T15:26:57Z
clusterinformations.crd.projectcalico.org     2018-06-08T15:26:57Z
felixconfigurations.crd.projectcalico.org     2018-06-08T15:26:57Z
globalnetworkpolicies.crd.projectcalico.org   2018-06-08T15:26:57Z
globalnetworksets.crd.projectcalico.org       2018-06-08T15:26:57Z
hostendpoints.crd.projectcalico.org           2018-06-08T15:26:57Z
ippools.crd.projectcalico.org                 2018-06-08T15:26:57Z
networkpolicies.crd.projectcalico.org         2018-06-08T15:26:57Z
~~~

If you go through the Calico documentation, you may come to know that Calico is mainly about two components: Datastore and Calico Felix agent. We are using kubernetes datastore for Calico, which means Felix directly learns the POD, NS and NetworkPolicies from kubernets instead of using etcd. In case of etcd informating is stored in etcd by Calico CNI plugin.

Typha is another important component in Calico world. Calico sits between Datastore and Felix. It's used to fetch the information from K8 datastore and feed that information to felix agents. Ths intermediator helps to reduce the load on datastore. Kindly refer [1] for more information. 

Default felix configuration file provides 172.17.0.0/12 subnet range which means that L2 bridge on each node will be configured from this range. Each node will be having route entries to provide connectivity to remote nodes. 

~~~
$ ip route list | tail
172.17.216.0/24 via 10.10.xx.xx dev tunl0 proto bird onlink
172.17.217.0/24 via 10.10.xx.xx dev tunl0 proto bird onlink
172.17.218.0/24 via 10.10.xx.xx dev tunl0 proto bird onlink
172.17.221.0/24 via 10.10.xx.xx dev tunl0 proto bird onlink
~~~

This route information is populated to all the nodes in setup using BGP.

[1] https://github.com/projectcalico/typha

