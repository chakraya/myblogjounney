---
layout: post
title: Create users on  Kubernetes cluster
description: How to create users on Kubernetes cluster using Role-based access control (RBAC)
comments_id: 2
---
{% seo %} 

## Introduction

In Kubernetes RBAC is used to manage user access to the  cluster resources  based on the roles which they are assigned to. To enable the RBAC in the cluster we need to start the Kubernetes API server by adding the RBAC option to the comma separated list as below.

```
--authorization-mode=Node,RBAC
```

RBAC is achieved through **rbac.authorization.k8s.io** API group. We can create custom policies dynamically via this API.

### **Types of roles**

Mainly there are 2 types of roles available. RBAC *Role* is used to grant access to namespaced specific  resources in contrast RBAC *ClusterRole* is used to grant access to cluster wide resources. There are many user authentication strategies such as X509 Client Certs , Static Token File ...etc. Here we are focusing only on X509 Client Certs strategy. When X509 is used, the Common Name (CN) of the certificate is used as username and Organization (O) field can be used to indicate the user’s group membership. In this post I will be discussing only on "Role".

As an example we will create a role which grants the **view logs in a specific pod** in the default namespace.

<img src="/assets/images/RBAC-X509.png" alt="">


#### **Creating a User**


```
openssl req -new -newkey rsa:2048 -nodes -out dev-user-readonly.csr -keyout dev-user-readonly.key -subj "/O=projectOne/CN=dev-user-readonly"
```
```
sudo openssl x509 -req -days 360 -in dev-user-readonly.csr -CA /var/lib/kubernetes/ca.crt  -CAkey /var/lib/kubernetes/ca.key -CAcreateserial -out dev-user-readonly.crt -sha256
```

*Once certificate is generated and signed by Kubernetes CA we can verify as below.*

```
openssl x509 -in dev-user-readonly.crt  --noout -text

Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number:
            46:a5:b3:9b:f4:bc:45:75:4d:7e:20:de:21:65:02:9f:f6:c9:bd:c6
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = KUBERNETES-CA
        Validity
            Not Before: Apr  9 16:36:14 2020 GMT
            Not After : Apr  4 16:36:14 2021 GMT
        Subject: O = projectOne, CN = dev-user-readonly
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
```




#### **What is inside the Role definition?**

Below are the short description of items used in YAML the file.

**apiGroups :** Set of REST endpoints which facilitate interaction with the cluster.

For more information [click here](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-groups)

Explore kubernetes API via kubectl proxy, this command needs to be run from the where you manage the cluster.

```sh
$ kubectl proxy --port=8090
```

We can list API endpoints using below CURL commands.
```sh
curl http://localhost:8090/apis
curl http://localhost:8090/api
```


**Resources :** Kubernetes objects such as pods , secrest , configmaps...etc. APIGroups are used to manage resources.

**Verbs :** Verbs are the actions which are available on resources.


#### **Creating the Role with restricted access.**

role.yaml
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: dev-user-readonly-role
rules:

- apiGroups: [""] # "" indicates the core API group
  resources: ["pods","pods/log"]
  resourceNames : ["tomcat-app1-7c57995bdc-2hbjx"] #Name of the POD
  verbs: ["get"]
```
```sh
$ kubectl create -f role.yaml
```
*Verify this Role creation.*
```sh
$ kubectl  describe role dev-user-readonly-role

Name:         dev-user-readonly-role
Labels:       <none>
Annotations:  <none>
PolicyRule:
  Resources  Non-Resource URLs  Resource Names                  Verbs
  ---------  -----------------  --------------                  -----
  pods/log   []                 [tomcat-app1-7c57995bdc-2hbjx]  [get]
  pods       []                 [tomcat-app1-7c57995bdc-2hbjx]  [get]

```

Next, we need to attach the Role created with the user. We use Rolebinding for this.

rolebinding.yaml
```
apiVersion: rbac.authorization.k8s.io/v1

# This role binding allows "dev-user-readonly" to read pods in the "default" namespace.
# You need to already have a Role named "dev-user-readonly-role" in that namespace.
kind: RoleBinding
metadata:
  name: dev-user-readonly-rolebinding
  namespace: default
subjects:
# You can specify more than one "subject"
# If we want to use Organization (O) defined in Cert (defined to use as user groups), "Kind" should be changed to "Group", So that all users in the defined group will get privileges.
- kind: User
  name: dev-user-readonly # "name" is case sensitive which is defined as CN in the user's certificate.
  apiGroup: rbac.authorization.k8s.io
roleRef:
  # "roleRef" specifies the binding to a Role / ClusterRole
  kind: Role #this must be Role or ClusterRole
  name: dev-user-readonly-role # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io

```

*Verify RoleBinding created*
```sh
$ kubectl  describe rolebindings dev-user-readonly-rolebinding

Name:         dev-user-readonly-rolebinding
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  Role
  Name:  dev-user-readonly-role
Subjects:
  Kind  Name               Namespace
  ----  ----               ---------
  User  dev-user-readonly
```


Finally we need to test the access by creating the required client config file. We use kubeconfig files to organize information about clusters, users, namespaces, and authentication mechanisms. The kubectl command-line tool uses kubeconfig files to find the information it needs to choose a cluster and communicate with the API server of a cluster. By default, kubectl looks for a file named config in the $HOME/.kube directory.


~/.kube/config-dev-user-readonly
```
apiVersion: v1
kind: Config
preferences: {}
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN0ekNDQVo4Q0ZFMGl4M2FvcUx3QTNRYW0vM2JOZXZUWUJnaHNNQTBHQ1NxR1NJYjNEUUVCQ3dVQU1CZ3gKRmpBVUJnTlZCQU1NRFV0VlFrVlNUa1ZVUlZNdFEwRXdIaGNOTWpBd05EQTBNRGt5T0RNM1doY05Nakl4TWpNdwpNRGt5T0RNM1dqQVlNUll3RkFZRFZRUUREQTFMVlVKRlVrNUZWRVZUTFVOQk1JSUJJakFOQmdrcWhraUc5dzBCCkFRRUZBQU9DQVE4QU1JSUJDZ0tDQVFFQXhoN1JjaFMxRVArTFR2ZzZyMnVoSXJpU0FYalI2YjR0SzNUcEY5RCsKZTJDaE9QTklTaXc1cmJFd2VWOGlRWENLQUpOakx2MnpjWFlYSTNSZU01SUJ4MHdwbHk1cE9XRzdGTzlyNkpMSwpGaHpndXRGRnFHK2s4cFBOYUxZM3VybHRndG9WR3hJcWFQZ05aYzRIMk1RdDRiTk5NRzlUeDZGVnJrZEZOUDlqCmU3a2h1OCtFN0krRDBjSnIwZlNSOENoWDljQ013eWtXcUVwc0pGN2dUN1ZQRm1aTFJTOGUxd1QxWlRrSDlSbEgKeUlwNXJkTlZUem42UVJML1FvbGFwbXBoSWYwWTdvWkJnaEpCeEM5SHlzYk1tMkZYaGFucDJ3Slhaejljc29ZbgpiamtiQlkvVHFvalVMK0tFb2cvTlk2Mlh5QmZ3Y29udjQxMUFWYUMzU3FoNXF3SURBUUFCTUEwR0NTcUdTSWIzCkRRRUJDd1VBQTRJQkFRQ1hTcWY2dUlXZUJKc242am1PRHlDOHJtWkpMdnRhU2RhSzY5c1gyUmpYb3Y5ejFpSHUKZzRBcnhxamxlN05xRVlGVXlrQytUMFduRnpQUS9XU0gzK2lEZ2dDZzU3N1hGcVNraWRPejY1RFU2MDFjK1BRKwpSeU1mSzJBYWJhRXdwUDdUeTdWRHlBTVk2Y2NnQ0tEczlWN2JuUkkzZ2hsZzV5OE5MZjZmNHgxTTQyb3pTdGVYCmYwdkxTNElTQjRXTTRFcUVjMlBJMUtqL0Z5UTBLTCtiY3RwQ05aMGk2b2pIOC8wNU9hYkRmVTlnUlBHaHlJTnEKREo5R3BTLzI0U01iT29KN2dwM1ZBNi9FWUJSWGkyRm13dzB5TU04Y1B6bHpKWFQ4QjZwUHU5UFlzMXZwS0YrNQpGenBIZVVnb09ZcC8wYTNEOVFrRTg3dmxNL09EcFhTUytGR1YKLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    server: https://192.168.5.30:6443 #API Server Endpoint
  name: dev-user-readonly-access

contexts:

- context:
    cluster: dev-user-readonly-access #Cluster Name
    user: dev-user-readonly #User name defined  during Cert creation as CN
  name: dev-user-readonly-context #Context name

current-context: dev-user-readonly-context

users:
- name: dev-user-readonly
  user:
    client-certificate: /home/vagrant/test/dev-user-readonly.crt
    client-key: /home/vagrant/test/dev-user-readonly.key
```


Let's see the POD logs now.

```sh
kubectl --kubeconfig ~/.kube/config-dev-user-readonly logs tomcat-app1-7c57995bdc-2hbjx -f

08-Apr-2020 23:36:38.618 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server version name:   Apache Tomcat/8.5.54
08-Apr-2020 23:36:38.621 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server built:          Apr 3 2020 14:06:10 UTC
08-Apr-2020 23:36:38.622 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server version number: 8.5.54.0
08-Apr-2020 23:36:38.622 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log OS Name:               Linux
08-Apr-2020 23:36:38.623 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log OS Version:            4.15.0-91-generic
08-Apr-2020 23:36:38.623 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Architecture:          amd64
08-Apr-2020 23:36:38.623 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Java Home:             /usr/local/openjdk-8/jre
08-Apr-2020 23:36:38.623 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log JVM Version:           1.8.0_242-b08
08-Apr-2020 23:36:38.624 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log JVM Vendor:            Oracle Corporation
08-Apr-2020 23:36:38.624 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log CATALINA_BASE:         /usr/local/tomcat
08-Apr-2020 23:36:38.624 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log CATALINA_HOME:         /usr/local/tomcat

```

If we try to view the logs of another POD which is not defined in the Role it will throw an error as below.
```sh
kubectl --kubeconfig ~/.kube/config-dev-user-readonly logs nginx-7cdbd8cdc9-28cf8  -f


Error from server (Forbidden): pods "nginx-7cdbd8cdc9-28cf8" is forbidden: User "dev-user-readonly" cannot get resource "pods" in API group "" in the namespace "default"
```

## Summary

RBAC is a very important feature in almost every enterprise environment. When dealing with different teams we require to have a different set of access levels defined for them. In this blog post I have described how to implement RBAC with the X509 Client Certs strategy.



