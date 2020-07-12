---
layout: post
title: How to use kubernetes client-go 
tags: [go, kubernetes]
category: [go, kubernetes]
author: vikrant
comments: true
--- 

This post will be covering information related to usage of [client-go](https://github.com/kubernetes/client-go).

- What is client-go?

Go client for talking to Kubernetes cluster. This is what kubectl used in background.

At the time of writing 10.0.0 is latest version. You may check [changelog](https://github.com/kubernetes/client-go/blob/master/CHANGELOG.md) for various versions. 

It's not necessary to use client-go, community provides library in other languages also like [python](https://github.com/kubernetes-client/python), java and c.. 

- What's the main dependancies of client-go?

go-client package depends upon [k8s.io/api](https://github.com/kubernetes/api) that is a collection of structs for kinds, and [k8s.io/apimachinery](https://github.com/kubernetes/apimachinery) that implements (group, version, resource) and (group, version, kind). 

#### Go example using client-go library.

I am using go version 1.13 but I am using old method of package management. 

~~~
$ go version
go version go1.13 linux/amd64

$ export GO111MODULE=on

$ cat list_k8s_nodes.go

package main

import (
        "flag"
        "fmt"
        "log"
        "os"
        "path/filepath"

        metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
        "k8s.io/client-go/kubernetes"
        "k8s.io/client-go/tools/clientcmd"
)

func main() {
        var ns string
        flag.StringVar(&ns, "namespace", "", "namespace")
        // Bootstrap k8s configuration from local       Kubernetes config file
        kubeconfig := filepath.Join(os.Getenv("HOME"), ".kube", "config")
        log.Println("Using kubeconfig file: ", kubeconfig)
        config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
        if err != nil {
                log.Fatal(err)
        }
        // Create an rest client not targeting specific API version
        clientset, err := kubernetes.NewForConfig(config)
        if err != nil {
                log.Fatal(err)
        }
        nodes, err := clientset.CoreV1().Nodes().List(metav1.ListOptions{})
        if err != nil {
                log.Fatalln("failed to get pods:", err)
        }
        // print nodes
        for i, node := range nodes.Items {
                fmt.Printf("[%d] %s\n", i, node.GetName())
        }
}

$ go mod init list_k8s_nodes.go  (This will create go.mod file in directory where go file is present)

Add the following in go.mod file. 

require (
	k8s.io/api kubernetes-1.13.2
	k8s.io/apimachinery kubernetes-1.13.2
	k8s.io/client-go v10.0.0
)

If you are hitting issue "package github.com/googleapis/gnostic/OpenAPIv2: cannot find package"

$ go get github.com/googleapis/gnostic@v0.4.0

$ go mod tidy

It should list the nodes present in k8s cluster.

$ go run list_k8s_nodes.go
~~~

If you wonder about usage of `clientset.CoreV1().Nodes().List(metav1.ListOptions{})` then please have a look at [this link](https://github.com/kubernetes/client-go/tree/master/kubernetes/typed/core/v1) specifically [for K8s nodes](https://github.com/kubernetes/client-go/blob/master/kubernetes/typed/core/v1/node.go) you will see various method associated with nodes struct out of these methods we have called LIST to get the list of nodes. 

#### Another method to query K8s API server

While searching about client-go I came across two option to query K8s API server:

- First which I shown above. 
- Second to cache the content on client side so that we reduce some load on API server

You may find good examples of client side caching on [1](https://github.com/vishal-biyani/kubernetes-days-india) and [2](https://github.com/alena1108/kubecon2017/blob/master/main.go).

While going through client-go repo if you wonder about dynamic directory then plese do read this [blog post](https://ymmt2005.hatenablog.com/entry/2020/04/14/An_example_of_using_dynamic_client_of_k8s.io/client-go) 