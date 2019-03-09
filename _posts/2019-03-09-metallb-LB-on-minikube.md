---
layout: post
title: How to run metallb on minikube setup?
tags: [Kubernetes]
category: [Kubernetes]
author: vikrant
comments: true
--- 

I am using minikube for a very long time. Whenever I created K8 service in minikube I always used Nodeport to access the service from external world but recently I came across interesting project metallb which we can use to provide the LB service on minikube, Virtual Machine K8 or baremetal setups. 

- Start the minikube VM. 

~~~
C:\Windows\system32>minikube start --memory 4096 --cpus 4
o   minikube v0.35.0 on windows (amd64)
>   Creating virtualbox VM (CPUs=4, Memory=4096MB, Disk=20000MB) ...
-   "minikube" IP address is 192.168.99.103
-   Configuring Docker as the container runtime ...
-   Preparing Kubernetes environment ...
-   Pulling images required by Kubernetes v1.13.4 ...
-   Launching Kubernetes v1.13.4 using kubeadm ...
:   Waiting for pods: apiserver proxy etcd scheduler controller addon-manager dns
-   Configuring cluster permissions ...
-   Verifying component health .....
+   kubectl is now configured to use "minikube"
=   Done! Thank you for using minikube!

C:\Windows\system32>minikube status
host: Running
kubelet: Running
apiserver: Running
kubectl: Correctly Configured: pointing to minikube-vm at 192.168.99.103
~~~

- Apply the metallb manifest. It will create a new NS, service account, RBAC, deployment and daemon for you. 

~~~
C:\Windows\system32>kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.7.3/manifests/metallb.yaml
namespace/metallb-system created
serviceaccount/controller created
serviceaccount/speaker created
clusterrole.rbac.authorization.k8s.io/metallb-system:controller created
clusterrole.rbac.authorization.k8s.io/metallb-system:speaker created
role.rbac.authorization.k8s.io/config-watcher created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:controller created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:speaker created
rolebinding.rbac.authorization.k8s.io/config-watcher created
daemonset.apps/speaker created
deployment.apps/controller created
~~~

- We can see the resources created by previous command in the following outputs. Important point to notice is that speaker is running in daemon mode which means it will run on all the nodes present in K8 setup however the controller will run as a replicaset on some of the nodes. 

~~~~
C:\Windows\system32>kubectl get ns
NAME             STATUS   AGE
default          Active   3m32s
kube-public      Active   3m27s
kube-system      Active   3m31s
metallb-system   Active   8s

C:\Windows\system32>kubectl get all -n metallb-system
NAME                              READY   STATUS    RESTARTS   AGE
pod/controller-7cc9c87cfb-g7vx8   1/1     Running   0          5m41s
pod/speaker-zrz58                 1/1     Running   0          5m42s

NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/speaker   1         1         1       1            1           <none>          5m42s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/controller   1/1     1            1           5m41s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/controller-7cc9c87cfb   1         1         1       5m41s
~~~

- Next we need to create the configuration file (configmap) for metallb in which we define the range of IPs from which IP addresses will be assigned to LB based services. 

Note: I am running metallb in L2 node, it's also possible to run it in BGP mode. 

~~~
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: custom-ip-pool
      protocol: layer2
      addresses:
      - 192.168.99.100/28
~~~ 

- Created test POD and service to test the functionality of metallb LB. 

~~~
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 80
  type: LoadBalancer
~~~

- Once the POD and service is successfully created, we should be able to see the ip address from the range assigned to my-service. 

~~~
C:\Users\DELL>kubectl get svc
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
kubernetes   ClusterIP      10.96.0.1      <none>          443/TCP          50m
my-service   LoadBalancer   10.110.13.91   192.168.99.96   8080:30396/TCP   10m
~~~

- Looking at the logs of controller and speaker when we created the service. 

Controller components is responsible for assigning the ip address to service. 

~~~
C:\Users\DELL>kubectl logs controller-7cc9c87cfb-g7vx8 -n metallb-system

{"caller":"service.go:88","event":"ipAllocated","ip":"192.168.99.96","msg":"IP address assigned by controller","service":"default/my-service","ts":"2019-03-09T08:15:21.353763822Z"}
~~~

- When we tried to access the page in browser, ARP request is generated to get the response from POD. 

~~~
C:\Users\DELL>kubectl logs speaker-zrz58 -n metallb-system

{"caller":"main.go:229","event":"serviceAnnounced","ip":"192.168.99.96","msg":"service has IP, announcing","pool":"custom-ip-pool","protocol":"layer2","service":"default/my-service","ts":"2019-03-09T08:15:21.374836442Z"}
{"caller":"main.go:231","event":"endUpdate","msg":"end of service update","service":"default/my-service","ts":"2019-03-09T08:15:21.374890323Z"}
{"caller":"arp.go:102","interface":"eth1","ip":"192.168.99.96","msg":"got ARP request for service IP, sending response","responseMAC":"08:00:27:1c:fa:33","senderIP":"192.168.99.1","senderMAC":"0a:00:27:00:00:3f","ts":"2019-03-09T08:15:45.16996067Z"}
~~~