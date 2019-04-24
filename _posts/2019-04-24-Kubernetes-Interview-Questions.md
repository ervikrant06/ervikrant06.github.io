---
layout: post
title: Kubernetes interview questions
tags: [Kubernetes]
category: [Kubernetes]
author: vikrant
comments: true
--- 

This article is under preparation.

From a long time, I was thinking about writing kubernetes interview questions, finally I found sometime to write on it. All the questions in this article are based on what I asked in interview, what I want to ask and what I was asked. 

Divided into following sections, I know these are overlapping sections it's difficult to distinguish between them:

- Administration
- Compute 
- Storage
- Network 
- Security
- Monitoring
- Logging 


#### Administration

Q: How to do maintenance activity on K8 node?

A: Maintenance activity are inevitable part of administration, you may need to do the patching or apply some security fixes on K8. Mark the node unschedulable and then drain the PODs which are present on K8 node.

 - kubectl cordon <hostname>
 - kubectl drain --ignore-daemonsets <hostname>

It's important to include the `--ignore-daemonsets` for any daemonset running on this node. Just in case if any statefulset is running on this node, and if no more node is available to maintain the count of statesful set then statesfulset POD will be in pending status. 

Q: What is role of a pause container?

A: Pause container servers as the parent container for all the containers in your POD. 

- It serves as the basis of Linux namespace sharing in the POD. 
- PID 1 for each POD to reap the zombie processes. 

https://www.ianlewis.org/en/almighty-pause-container

Q: How will you update the version of K8?

A: Before doing the update of K8, it's important to read the release notes to understand the changes introduced in newer version and whether version update will also update the etcd. 

https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade-1-12/

Q: Difference between helm and K8 operator?

A: An Operator is an application-specific controller that extends the Kubernetes API to create, configure and manage instances of complex stateful applications on behalf of a Kubernetes user. It builds upon the basic Kubernetes resource and controller concepts, but also includes domain or application-specific knowledge to automate common tasks better managed by computers. On the other hand, helm is a package manager like yum or apt-get. 

Q: Explain the role of CRD (Custom Resource Definition) in K8?

A: A custom resource is an extension of the Kubernetes API that is not necessarily available in a default Kubernetes installation. It represents a customization of a particular Kubernetes installation. However, many core Kubernetes functions are now built using custom resources, making Kubernetes more modular.

#### Compute

Q: How to troubleshoot if the POD is not getting scheduled? 

A: There are many factors which can led to unstartable POD. Most common one is running out of resources, use the commands like `kubectl desribe <POD> -n <Namespace>` to see the reason why POD is not started. Also, keep an eye on `kubectl get events` to see all events coming from the cluster. 

#### Storage

Q: How to provide persistent storage for POD?

A: Persistent volumes are used for persistent POD storage. They can be provision statically or dynamically. 

Static : A cluster administrator creates a number of PVs. They carry the details of the real storage which is available for use by cluster users. 

Dynamically : Administrator creates a PVC (Persistent volume claim) specifying the existing storage class and volume created dynamically based on PVC.

#### Network

Q: How two containers running in a single POD have single IP address?

A: Kubernetes implements this by creating a special container for each pod whose only purpose is to provide a network interface for the other containers. These is one `pause` container which is responsible for namespace sharing in the POD. Generally, people ignore the existance of this pause container but actually this container is the heart of network and other functionalities of POD. It provides a single virtual interface which is used by all containers running in a POD.  

Q: What are the various ways to provide external world connectivity to K8?

A: By default POD should be able to reach the external world but for vice-versa we need to do some work. Following options are available to connect with POD from outer world. 

- Nodeport (it will expose one port on each node to communicate with it)
- Load balancers (L4 layer of TCP/IP protocol)
- Ingress (L7 layer of TCP/IP Protocol)

One another method is kube-proxy which can be used to expose a service with only cluster IP on local system port. 

$ kubectl proxy --port=8080
$ http://localhost:8080/api/v1/proxy/namespaces/<NAMESPACE>/services/<SERVICE-NAME>:<PORT-NAME>/

https://medium.com/google-cloud/kubernetes-nodeport-vs-loadbalancer-vs-ingress-when-should-i-use-what-922f010849e0

Q: What's the difference between nodeport and load balancer?

A: nodport relies on the IP address of your node. Also, you can use the node ports only from the range 30000–32767, on another hand load balancer will have it's own IP address. All the major cloud providers supports creating the LB for you if you specify LB type while creating the service. On baremetal based clusters, metallb is promising.  

Q: When we need ingress instead of a LB? 

A: For each service you have one LB. You can have single ingress for multiple services. This will allow you do both path based and subdomain based routing to backend services. You can do the SSL termination at ingress. 

Q: How POD to POD communication works?

A: For POD to POD communication, it's always recommended to use the K8 service DNS instead of POD IP because PODs are ephemeral and their IPs can get change after the redeployment. 

If the two PODs are running on a same host then physical interface will not come into the picture. 

- Packet will leave POD1 virtual network interface and go to docker bridge (cbr0).
- Docker bridge will forward the packet to right POD2 which is running on same host.  

If two PODs are running on a different host then physical interface of both host machines will come into the picture. Let's consider a scenario in which CNI is not used. 

POD1 = 192.168.2.10/24 (node1, cbr0 192.168.2.1)
POD2 = 192.168.3.10/24 (node2, cbr1 192.168.3.1)

- POD1 will send the traffic destined for POD2 to it's GW (cbr0) because both are in different subnet. 
- GW doesn't know about 192.168.3.0/24 network hence it will forward the traffic to physical interface of node1. 
- node1 will forward the traffic to it's own physical rourter/gateway.
- That physical router/GW should have the route for 192.168.3.0/24 network to route the traffic to node2.
- Once traffic reaches node2, it pass that traffic to POD2 through cbr1

If the Calico CNI it's responsible for adding the routes for cbr (docker bridge IP address) in all nodes. 

Q: How POD to service communication works?

- PODs are ephemeral their IP address can change hence to communicate with POD in reliable way service is used as a proxy or load balancer. A service is a type of kubernetes resource that causes a proxy to be configured to forward requests to a set of pods. The set of pods that will receive traffic is determined by the selector, which matches labels assigned to the pods when they were created. K8 provides an internal cluster DNS that resolves the service name. 

Service is using different internal network than POD network. netfilter rules which are injected by kube-proxy are used to redirect the request actually destined for service IP to right POD. 



I highly recommend reading the following series to get solid understanding about the K8 networking. 

https://medium.com/google-cloud/understanding-kubernetes-networking-pods-7117dd28727

https://medium.com/google-cloud/understanding-kubernetes-networking-services-f0cb48e4cc82

https://medium.com/google-cloud/understanding-kubernetes-networking-ingress-1bc341c84078


#### Security

Q: What are the various things can be done to increase the K8 security?

A: This is a huge topic, I am sharing some thoughts on it.

- By default, POD can communicate with any other POD, we can setup network policies to limit this communication between the PODs.
- RBAC (Role based access control) to narrow down the permissions. 
- Use namespaces to establish security boundaries. 
- Set the admission control policies to avoid running the priviledged containers. 
- Turn on audit logging.

#### Monitoring

Q: How to monitor K8 cluster? 

A: Prometheus is used for K8 monitoring. Prometheus ecosystem consists of multiple components. 

- main Prometheus server which scrapes and stores time series data
- client libraries for instrumenting application code
- a push gateway for supporting short-lived jobs
- special-purpose exporters for services like HAProxy, StatsD, Graphite, etc.
- an alertmanager to handle alerts
- various support tools

Q: How to make prometheus HA?

A: You may run multiple instances of prometheus HA but grafana can use only of them as a datasource. You may put load balancer in front of multiple prometheus instances, use sticky sessions and failover if one of the prometheus instance dies. This make things complicated. Thanos is another project which solve these challenges. 

Q: What are other challenges with prometheus?

A: Desipte of being very good at K8 monitoring, prometheus still have some issues: 

- Prometheus HA support.
- No downsampling is available for collected metrics over the period of time. 
- No support for object storage for long term metric retention.

All of these challenges are again overcome by Thanos.   

Q: What's prometheus operator?

A: The mission of the Prometheus Operator is to make running Prometheus on top of Kubernetes as easy as possible, while preserving configurability as well as making the configuration Kubernetes native.

#### Logging

Q: How to get the central logs from POD?

A: This architecture depends upon application and many other factors. Following are the common logging patters

- Node level logging agent
- Streaming sidecar container
- Sidecar container with logging agent
- Export logs directly from the application

In our setup, filebeat and journalbeat are running as daemonset. Logs collected by these are dumped to kafka topic which are eventually dumped to ELK stack. 

Same can be achieved using EFK stack and fluentd-bit.   

