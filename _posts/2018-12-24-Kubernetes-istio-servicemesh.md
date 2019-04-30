---
layout: post
title: Istio (Service Mesh) 101
tags: [Istio, Kubernetes]
category: [Istio, Kubernetes]
author: vikrant
comments: true
--- 

If you are working in kubernetes world then you should have heard the word Istio, if not Istio then at-least service mesh. I heard about istio couple of times, I tried it sometime back in my lab environment but didn't get the chance to look into it more closely. Today, I found sometime to dig deep into it and try out some example. In this post, I am sharing the basic exercise which I completed on my minikube setup. In future posts, we will dig more on istio topic. 

#### What is istio?

Istio is a service mesh which helps you control, observe, secure and control services E/W traffic mainly. You may think that we can achieve the same using K8 networking layer which a already provides similar features, if it doesn't then same can be introduced in your app. Your app can export prometheus formatted stats, it can use OpenTracer to introduce distributed tracing capabilities, and to secure traffic mTLS can be introduced then why we need it?

Istio decouples all the points mentioned above from your service. You have sidecar proxy container running along with service which can do everything for you, reducing the pain of app developer. Istio is a control plane for envoy proxy dataplane. Envoy is a proxy created by Lyft to solve the internal challenges with large scale K8 setup. 

#### Setup Information:

- Minikube running with kubernetes version v1.14.0 with 5GB RAM and 4 CPUs.

- Deployed istio following the instructions from official [istio documentation)(https://istio.io/docs/setup/kubernetes/install/kubernetes/). You should see following PODs in `istio-system` namespace. 

~~~
$ kubectl get pod -n istio-system
NAME                                      READY   STATUS      RESTARTS   AGE
grafana-67c69bb567-s25fw                  1/1     Running     3          2d5h
istio-citadel-78c9c8b75f-6srt4            1/1     Running     3          2d5h
istio-cleanup-secrets-1.1.4-g7m26         0/1     Completed   0          2d5h
istio-egressgateway-6df84c5bd4-7r8gh      1/1     Running     3          2d5h
istio-galley-65fc98ffd4-ltstm             1/1     Running     2          32h
istio-grafana-post-install-1.1.4-qhd6g    0/1     Completed   0          2d5h
istio-ingressgateway-78f9cbb78f-tvhzk     1/1     Running     3          2d5h
istio-pilot-6b75486f59-czlkv              2/2     Running     6          2d5h
istio-policy-784c66bc85-cbn9z             2/2     Running     16         2d5h
istio-security-post-install-1.1.4-s7ghg   0/1     Completed   0          2d5h
istio-sidecar-injector-bf946798-252v4     1/1     Running     9          2d5h
istio-telemetry-8476d56f55-b45mv          2/2     Running     16         2d5h
istio-tracing-5d8f57c8ff-9h5mz            1/1     Running     3          2d5h
kiali-d4d886dd7-5lzsh                     1/1     Running     2          33h
prometheus-5554746896-s7v24               1/1     Running     3          2d5h
~~~

`istio-pilot` and `istio-telemetry` are two specical PODs with two containers, rest all are having one container in POD. 

~~~
$ kubectl logs -f istio-pilot-6b75486f59-czlkv -n istio-system -c discovery
~~~


- `istio` repo comes with some default examples, famous one is bookinfo to keep the things simple for a new learner, I started with helloworld example instead. Manifest will container two helloworld deployment with different version labels and one service. 

~~~
apiVersion: v1
kind: Service
metadata:
  name: helloworld
  labels:
    app: helloworld
spec:
  ports:
  - port: 5000
    name: http
  selector:
    app: helloworld
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: helloworld-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: helloworld
        version: v1
    spec:
      containers:
      - name: helloworld
        image: istio/examples-helloworld-v1
        resources:
          requests:
            cpu: "100m"
        imagePullPolicy: IfNotPresent #Always
        ports:
        - containerPort: 5000
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: helloworld-v2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: helloworld
        version: v2
    spec:
      containers:
      - name: helloworld
        image: istio/examples-helloworld-v2
        resources:
          requests:
            cpu: "100m"
        imagePullPolicy: IfNotPresent #Always
        ports:
        - containerPort: 5000
~~~

Use the following command to deploy it. This command will insert istio-proxy containers as a sidecar or you can make the sidecar injecting default. 

~~~
$ kubectl create -f <(istioctl kube-inject -f samples/helloworld/custom_helloworld.yaml)
~~~

Following two deployments and service should be present in default namespace. 

~~~
$ kubectl get pod -o wide
NAME                             READY     STATUS    RESTARTS   AGE       IP            NODE
helloworld-v1-575d9d9fcd-vgcx6   2/2       Running   0          1h        172.17.0.14   minikube
helloworld-v2-54b97b8585-v6hbv   2/2       Running   0          1h        172.17.0.3    minikube

$ kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
helloworld   ClusterIP   10.103.57.51   <none>        5000/TCP    8h
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    2h
~~~

- Deploy the curl POD which will be used to call the helloworld app. I deployed another POD with curl1. You just need to replace curl with curl1 in below manifest everywhere. 

~~~
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: curl-deploy
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: curl
    spec:
      containers:
      - name: curl
        image: radial/busyboxplus:curl
        imagePullPolicy: Always
        command:
        - sh
        - -c
        - while true; do sleep 1; done
---
apiVersion: v1
kind: Service
metadata:
  name: curl
  labels:
    app: curl
spec:
  ports:
  - port: 15001
    name: curlport
  selector:
    app: curl
~~~        

Again use the istioctl to inject the sidecare istio-proxy. 

~~~
kubectl create -f <(istioctl kube-inject -f samples/helloworld/curl-deploy.yaml)
~~~ 

We should have following PODs running after following above steps. 

~~~
$ kubectl get pod --show-labels
NAME                             READY   STATUS    RESTARTS   AGE    LABELS
curl-deploy-fb5d48bb8-q8l57      2/2     Running   0          91m    app=curl,pod-template-hash=fb5d48bb8
curl-deploy1-5df76f78bb-bzw4t    2/2     Running   0          91m    app=curl1,pod-template-hash=5df76f78bb
helloworld-v1-5dbccc9889-75hhv   2/2     Running   0          123m   app=helloworld,pod-template-hash=5dbccc9889,version=v1
helloworld-v2-585b7495cc-6ffhf   2/2     Running   0          123m   app=helloworld,pod-template-hash=585b7495cc,version=v2
~~~ 

- Connect with terminal of both curl pods and issue curl command couple of times to see the RR response from helloworld service. 

~~~
$ kubectl exec -it curl-deploy-5f7684bb65-468vt -c curl sh

[ root@curl-deploy-5f7684bb65-468vt:/ ]$ curl helloworld.default.svc.cluster.local:5000/hello
Hello version: v1, instance: helloworld-v1-575d9d9fcd-vgcx6

[ root@curl-deploy-5f7684bb65-468vt:/ ]$ curl helloworld.default.svc.cluster.local:5000/hello
Hello version: v2, instance: helloworld-v2-54b97b8585-v6hbv
~~~

- Using istio DestinationRule CRD. Create destination rule which will map the version v1 and v2 of helloworld app with subsets which are understandable by istio. Using subset v1 for helloworld version v1 and same for v2. Redirect the traffic coming from POD with label `app: curl` to v1 version of helloworld. Traffic coming from `app: curl1` will be redirected to both versions of helloworld.  

~~~
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld-virtualservice
spec:
  hosts:
  - "helloworld"
  http:
  - match:
    - sourceLabels:
        app: curl
    route:
    - destination:
        host: helloworld
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: helloworld-destinationrule
spec:
  host: helloworld
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
~~~

- Curl from POD with label `app: curl` will always get a response from the v1 version of helloworld, however the curl1 running with label `app: curl1` will get response from both v1 and v2 versions of helloworld. 

~~~
[ root@curl-deploy-5f7684bb65-468vt:/ ]$ curl helloworld.default.svc.cluster.local:5000/hello
Hello version: v1, instance: helloworld-v1-575d9d9fcd-vgcx6
~~~
