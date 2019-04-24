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

 kubectl cordon <hostname>
 kubectl drain --ignore-daemonsets <hostname>

It's important to include the `--ignore-daemonsets` for any daemonset running on this node. Just in case if any statefulset is running on this node, and if no more node is available to maintain the count of statesful set then statesfulset POD will be in pending status. 

Q: What is role of a pause container?

A: Pause container servers as the parent container for all the containers in your POD. 

- It serves as the basis of Linux namespace sharing in the POD. 
- PID 1 for each POD to reap the zombie processes. 

https://www.ianlewis.org/en/almighty-pause-container


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


