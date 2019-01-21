---
layout: post
title: How to start multi-node K8 setup in containers?
tags: [Kubernetes, dind]
category: [Kubernetes]
author: vikrant
comments: true
--- 

Running in single node kubernetes setup using minikube is a very easy task, simply issue a command, BOOM, your single node setup is up. Running a multi-node kubernetes setup is a bit of task. I was looking for a easy way to run the multi node setup with minimal configuration. We do have options like creating VM using vagrant and then using ansible playbooks for the installation. But I was looking for more quicker approach ;) I came across this [interesting project](https://github.com/kubernetes-sigs/kubeadm-dind-cluster) in which we can run the multi-node K8 setup inside the docker containers. This is possible because of DinD (Docker in docker), if you want to read about it refer my [last post](https://ervikrant06.wordpress.com/2016/12/24/dind-docker-in-docker-on-atomic-host/) on my old blog.  

It's very easy to start a cluster running a single shell script. I installed the latest K8 1.13 version. 

- Three kubernetes nodes are running with default shell script. Among three, one is master and other two are worker/minion nodes. 

~~~
$ kubectl get nodes
NAME          STATUS    ROLES     AGE       VERSION
kube-master   Ready     master    3h        v1.13.0
kube-node-1   Ready     <none>    3h        v1.13.0
kube-node-2   Ready     <none>    3h        v1.13.0

$ kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health": "true"}
~~~

- Corresponding to above three K8 nodes we can see three docker containers running. 

~~~
$ docker ps
CONTAINER ID        IMAGE                                                                          COMMAND                  CREATED             STATUS              PORTS                       NAMES
b5c603cdfae0        mirantis/kubeadm-dind-cluster:596f7d093470c1dc3a3e4466bcdfb34438a99b90-v1.13   "/sbin/dind_init sys…"   3 hours ago         Up 3 hours          8080/tcp                    kube-node-2
43a7e25b1a30        mirantis/kubeadm-dind-cluster:596f7d093470c1dc3a3e4466bcdfb34438a99b90-v1.13   "/sbin/dind_init sys…"   3 hours ago         Up 3 hours          8080/tcp                    kube-node-1
89f63c617b7c        mirantis/kubeadm-dind-cluster:596f7d093470c1dc3a3e4466bcdfb34438a99b90-v1.13   "/sbin/dind_init sys…"   3 hours ago         Up 3 hours          127.0.0.1:32778->8080/tcp   kube-master
~~~

- Run test POD in kubernetes cluster using the command. This will start POD inside the docker container. If you issue `docker ps` command again only three containers will be thr. 

~~~
$ kubectl run -i --tty busybox --image=busybox --restart=Never -- sh
If you don't see a command prompt, try pressing enter.
~~~

- Let's check where this POD is running. It's running in minion node, issue the `docker ps` command inside the docker container to see the running container corresponding to POD. 

~~~
$ kubectl get pod -o wide
NAME      READY     STATUS    RESTARTS   AGE       IP           NODE
busybox   1/1       Running   0          25m       10.244.3.4   kube-node-2

$ docker exec -it b5c603cdfae0 docker ps | grep busybox
94a8ebdf2b94        busybox                "sh"                     25 minutes ago      Up 25 minutes                           k8s_busybox_busybox_default_4dece815-1d73-11e9-a5d4-be9163280214_0
f13bcff2bfe3        k8s.gcr.io/pause:3.1   "/pause"                 25 minutes ago      Up 25 minutes                           k8s_POD_busybox_default_4dece815-1d73-11e9-a5d4-be9163280214_0
~~~

- Our busybox POD is able to ping the external world, gateway of POD is present on worker node but where the worker node gateway is present. In case of MAC, dockers are running inside the VM. You can find the gateway of worker node gateway inside the LinuxKit VM.

~~~ 
/ # ip route list
default via 10.244.3.1 dev eth0
10.244.3.0/24 dev eth0 scope link  src 10.244.3.4

/ # ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=36 time=91.981 ms
^C
--- 8.8.8.8 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 91.981/91.981/91.981 ms
/ #

$ docker exec -it b5c603cdfae0 ifconfig dind0
dind0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.244.3.1  netmask 255.255.255.0  broadcast 0.0.0.0
        ether fe:01:92:eb:36:b9  txqueuelen 1000  (Ethernet)
        RX packets 22025  bytes 1419409 (1.3 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 23188  bytes 8447136 (8.0 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

$ docker run --privileged --pid=host -it --rm ubuntu nsenter -t 1 -m -u -i -n -p /bin/sh
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 brd 127.255.255.255 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 02:50:00:00:00:01 brd ff:ff:ff:ff:ff:ff
    inet 192.168.65.3/24 brd 192.168.65.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::50:ff:fe00:1/64 scope link
       valid_lft forever preferred_lft forever
3: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN qlen 1
    link/ipip 0.0.0.0 brd 0.0.0.0
4: ip6tnl0@NONE: <NOARP> mtu 1452 qdisc noop state DOWN qlen 1
    link/tunnel6 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00 brd 00:00:00:00:00:00:00:00:00:00:00:00:00:00:00:00
6: docker_gwbridge: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:06:1b:dd:b7 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global docker_gwbridge
       valid_lft forever preferred_lft forever
    inet6 fe80::42:6ff:fe1b:ddb7/64 scope link
       valid_lft forever preferred_lft forever
7: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:d8:a9:ff:f3 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:d8ff:fea9:fff3/64 scope link
       valid_lft forever preferred_lft forever
13: vethca82746@if12: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue master docker_gwbridge state UP
    link/ether 3a:9d:20:a2:59:a9 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::389d:20ff:fea2:59a9/64 scope link
       valid_lft forever preferred_lft forever
14: cni0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN qlen 1000
    link/ether a6:ec:f9:4d:2e:44 brd ff:ff:ff:ff:ff:ff
    inet 10.1.0.1/16 scope global cni0
       valid_lft forever preferred_lft forever
    inet6 fe80::a4ec:f9ff:fe4d:2e44/64 scope link
       valid_lft forever preferred_lft forever
59: br-9874377b5e7e: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
    link/ether 02:42:cd:08:3d:47 brd ff:ff:ff:ff:ff:ff
    inet 10.192.0.1/24 brd 10.192.0.255 scope global br-9874377b5e7e
       valid_lft forever preferred_lft forever
    inet6 fe80::42:cdff:fe08:3d47/64 scope link
       valid_lft forever preferred_lft forever
67: veth176b0e5@if66: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue master br-9874377b5e7e state UP
    link/ether b2:9c:37:fd:79:53 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::b09c:37ff:fefd:7953/64 scope link
       valid_lft forever preferred_lft forever
69: vethf20841c@if68: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue master br-9874377b5e7e state UP
    link/ether 2a:b3:5a:56:af:48 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::28b3:5aff:fe56:af48/64 scope link
       valid_lft forever preferred_lft forever
71: vetha9b9aac@if70: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue master br-9874377b5e7e state UP
    link/ether 32:b6:49:af:cd:f6 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::30b6:49ff:feaf:cdf6/64 scope link
       valid_lft forever preferred_lft forever
73: veth4d81314@if72: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue master docker0 state UP
    link/ether 26:68:44:97:c4:34 brd ff:ff:ff:ff:ff:ff
    inet6 fe80::2468:44ff:fe97:c434/64 scope link
       valid_lft forever preferred_lft forever
/ #
~~~

