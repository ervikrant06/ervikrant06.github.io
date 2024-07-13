---
layout: post
title: How to use kyverno to replicate k8s resources between namespaces
tags: [k8s, kyverno, kubernetes]
category: [k8s, kyverno, kubernetes]
author: Vikrant
comments: true
---

This article provide the steps to use kyverno to replicate the resources between Hierarchical namespaces (HNC) using kyverno.

## HNC namespace hierarchy

~~~
$ kubectl hns tree test-root

test-root
└── test
    ├── [s] test-qa
    │   ├── [s] test-qa-apps
    │   ├── [s] test-qa-docs
    │   └── [s] test-qa-monitoring
~~~

Example scenario: ConfigMap from namespace test-qa-apps needs to be replicated in test-root namespace

## Kyerno policy

#### What this policy is doing

This policy is finding `ConfigMap` resource with name `test1` in `test-qa-apps` namespace.
If the resource is found then create a `ConfigMap` resource in `test-root` namespace. HNC root namespace name is deduced through the labels on `test-qa-apps` namespace

Create a kyverno clusterpolicy:

~~~yaml
cat <<EOF > kyerno_cluster_policy.yml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: test-test
spec:
  background: true
  rules:
  - context:
    - apiCall:
        method: GET
        urlPath: /api/v1/namespaces/{{ request.object.metadata.namespace }}
      name: namespace
    generate:
      apiVersion: v1
      data:
        metadata:
          annotations: "{{ \tmerge(request.object.metadata.annotations || `{}`, {
            \t\t\"origin-name\": request.object.metadata.name,
            \t\t\"origin-namespace\": request.object.metadata.namespace
            \t}) }}"
        data:
          commonName: '{{ request.object.data.test }}'
      kind: ConfigMap
      name: kyverno-{{ request.object.metadata.namespace }}-{{ request.object.metadata.name }}
      namespace: "{{ \tmax_by( items(namespace.metadata.labels, 'label', 'value')[?label.ends_with(@,
        '.tree.hnc.x-k8s.io/depth')], &value ). \tlabel.replace_all(@, '.tree.hnc.x-k8s.io/depth',
        '') }}"
      synchronize: true
    match:
      any:
      - resources:
          kinds:
          - ConfigMap
          name: test1
          namespaceSelector:
            matchExpressions:
            - key: kubernetes.io/metadata.name
              operator: In
              values:
              - "test-qa-apps"
    name: test-test-resource
EOF
~~~

Apply the policy

~~~
$ kubectl apply -f kyerno_cluster_policy.yml
~~~

#### Explanation of policy

-  This namespace variable contains the name of namespace where the resource is detected. In this case `test-qa-apps` namespace

~~~yaml
    - apiCall:
        method: GET
        urlPath: /api/v1/namespaces/{{ request.object.metadata.namespace }}
      name: namespace
~~~

- Let's check the test-qa-apps namespace labels to understand how `test-root` namespace name is deduced from it

Truncated output:
~~~yaml
$ kubectl get ns test-qa-apps -oyaml
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    hnc.x-k8s.io/subnamespace-of: test-qa
  labels:
    test-qa-apps.tree.hnc.x-k8s.io/depth: "0"
    test-qa.tree.hnc.x-k8s.io/depth: "1"
    test-root.tree.hnc.x-k8s.io/depth: "3"
    test.tree.hnc.x-k8s.io/depth: "2"
    kubernetes.io/metadata.name: test-qa-apps
  name: test-qa-apps
~~~

To deduce `test-root` i.e destination namespace from the source namespace, we need to get the lable with maximum depth and then pick the value of that label.

~~~yaml
      namespace: "{{ \tmax_by( items(namespace.metadata.labels, 'label', 'value')[?label.ends_with(@,
        '.tree.hnc.x-k8s.io/depth')], &value ). \tlabel.replace_all(@, '.tree.hnc.x-k8s.io/depth',
        '') }}"
~~~


#### Test working

~~~
$ kubectl create cm test1 --from-literal=test=newvalue -n test-qa-apps
configmap/test1 created

$ kubectl get cm kyverno-test-qa-apps-test1  -n test-root
NAME                        DATA   AGE
kyverno-test-qa-apps-test1   1      45s
~~~

Check the definition of resource

~~~yaml
$ kubectl get cm kyverno-test-qa-apps-test1  -n test-root -oyaml
apiVersion: v1
data:
  commonName: newvalue
kind: ConfigMap
metadata:
  annotations:
    origin-name: test1
    origin-namespace: test-qa-apps
  labels:
    app.kubernetes.io/managed-by: kyverno
    generate.kyverno.io/policy-name: test-test
    generate.kyverno.io/policy-namespace: ""
    generate.kyverno.io/rule-name: test-test-resource
    generate.kyverno.io/trigger-group: ""
    generate.kyverno.io/trigger-kind: ConfigMap
    generate.kyverno.io/trigger-name: test1
    generate.kyverno.io/trigger-namespace: test-qa-apps
    generate.kyverno.io/trigger-version: v1
  name: kyverno-test-qa-apps-test1
  namespace: test-root
~~~

## Important points

- If the kyverno policy is deleted then the resources created with the help of policy are also deleted. Ex: `kyverno-test-qa-apps-test1` will be deleted
- Any changes in source ConfigMap `test1` reflects in `kyverno-test-qa-apps-test1`

## What's the practical use case

- This post is motivated from my colleague awesome work. We had a requirement of using cert-manager along with HNC, because of some restrictions we have to create the Certificate resource in root level 
namespace hence we created a CRD (Custom Resource defintion) which end-users can use to create the resources in their managed namespaces. Kyverno policy is used to replicate this resource to root namespace 
then secret created by cert-manger is propogated back to user namespace.
