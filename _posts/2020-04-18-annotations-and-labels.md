---
layout: post
title: Kubernetes Labels and Annotations
description: Discuss the usage of kubernetes Labels and Annotations
comments_id: 3
---

### **Labels**

In Kubernetes Labels are used to identify/search objects. Some of the usages are to group pods for a deployment. So labels contain identifying information which can be used by selectors. 

*Ex:*

Below is the tomcat pod running with a label set to **run=tomcat-app1**. We can use a selector to select (identify)  only that pod.

*Without selector*
```sh
$ kubectl  get pods

NAME                           READY   STATUS    RESTARTS   AGE
tomcat-app1-7c57995bdc-2hbjx   1/1     Running   0          33h
tomcat-app2-c9bd88677-m2x9h    1/1     Running   0          33h
```

*With selector*
```sh
$ kubectl  get pods -l run=tomcat-app1 (-l, --selector='': Selector (label query) to filter on, supports '=', '==', and '!='.(e.g. -l key1=value1,key2=value2))

NAME                           READY   STATUS    RESTARTS   AGE
tomcat-app1-7c57995bdc-2hbjx   1/1     Running   0          33h
```

Inside the **tomcat-app1-7c57995bdc-2hbjx** definition , we have the label **run=tomcat-app1** as below.

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-04-08T23:36:25Z"
  generateName: tomcat-app1-7c57995bdc-
  labels:
    pod-template-hash: 7c57995bdc
    run: tomcat-app1
……
```

Further in the deployments , deployment definition uses the labels to  select the pods which deployment definition should  be applied.

*Ex:*
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  creationTimestamp: "2020-04-08T23:36:25Z"
  generation: 1
  labels:
    run: tomcat-app1
  name: tomcat-app1
…..
```

### **Annotations**

On the other hand annotations are used to store information which is useful for later  inspections. Those  are non-identifying metadata. Some of the examples are as below. 
Using this **kubernetes.io/change-cause** annotation we can store the last commands executed on the given object, which can be useful for future inspections.

*Creating nginx pod*
```sh
$ kubectl run --generator=run-pod/v1  nginx-pod --image=nginx:1.15 
```
*Now let’s update the pod’s nginx version.*
```sh
$ kubectl set image pod/nginx-pod  nginx-pod=nginx:1.16 --record  ( Use  the –record flag to write the command executed in the resource annotation kubernetes.io/change-cause )
```
```sh
$ kubectl  describe  pod nginx-pod

Name:               nginx-pod
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               worker-2/192.168.5.22
Start Time:         Fri, 10 Apr 2020 08:59:13 +0000
Labels:             run=nginx-pod
Annotations:        kubernetes.io/change-cause: kubectl set image pod/nginx-pod nginx-pod=nginx:1.16 --record=true
Status:             Running
```

Also we can use the **annotate** command to add custom annotations as below.
```sh
$ kubectl  annotate pod  nginx-pod email=myemail@gmail.com
```
```sh
$ kubectl  describe  pod nginx-pod

Name:               nginx-pod
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               worker-2/192.168.5.22
Start Time:         Fri, 10 Apr 2020 08:59:13 +0000
Labels:             run=nginx-pod
Annotations:        email: myemail@gmail.com
                    kubernetes.io/change-cause: kubectl set image pod/nginx-pod nginx-pod=nginx:1.16 --record=true
```

## Summary

Labels are used to identify objects and in contrast annotations are used to store various information for various purposes.
