---
layout: post
title: Using Kubectl Proxy for Devops Purposes
comments_id: 4
---
{% seo %}

## Access Kubernetes Cluster services from outside for debugging purposes.

Kubernetes Cluster services (ClusterIP type) are accessed only within the cluster. But there are situations where we need to access those  from devops machines for continuous testing and developments.

First install “kubectl” command in the devops laptop as below.

```sh
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
```
```sh
$ chmod 700 ./kubectl
```

Now you can use the existing kubeconfig file or create a new one according to the privileges you required. Refer this [link](https://chakraya.com/2020/04/12/configure-rbac-in-kubernetes.html) for creating new users with required access rights. Place the config file on “~/.kube/config” and **kubectl** will automatically pick that as the config file to use.





*Checking the access*
```sh
$ ./kubectl  get svc

NAME                  TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
kubernetes            ClusterIP   10.96.0.1     <none>        443/TCP          5d13h
my-service            ClusterIP   10.96.0.57    <none>        80/TCP           5h24m
nginx                 NodePort    10.96.0.129   <none>        80:32138/TCP     6d8h
tomcat-nodeport-svc   NodePort    10.96.0.137   <none>        8080:30036/TCP   5d14h
tomcat-service-app1   ClusterIP   10.96.0.79    <none>        80/TCP           44h
tomcat-service-app2   ClusterIP   10.96.0.76    <none>        80/TCP           44h


```


Now let's see how we can access the cluster services using **kubectl proxy** command.

```sh
$ ./kubectl proxy

Starting to serve on 127.0.0.1:8001

```
Now you can use **127.0.0.1:8001** to access internal **ClusterIP** type services for debugging purposes.

**One more important thing is**, if we are not using a master node (API server node) as a worker node we should add a route for POD network traffic. Here I have added a route to direct the POD network traffic to one of the worker nodes.

When this route is not present you will get an error as follows when trying to access ClusterIP service via kubectl proxy.

```
Error: 'dial tcp 10.32.0.4:8080: i/o timeout

```



*On master node add the following route entry*
```sh
$ route add -net 10.32.0.0/12 gw 192.168.5.21

10.32.0.0/12 (POD network range)
192.168.5.21 (worker node IP)
```

We can access the service in the following way.

```sh
http://kubernetes_master_address/api/v1/namespaces/namespace_name/services/service_name[:port_name]/proxy
```
[More Info](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/)

In my example URL would be as follows.

```sh
http://127.0.0.1:8001/api/v1/namespaces/default/services/tomcat-service-app1/proxy/sample/hello.jsp

```
<img src="/assets/images/kubectl-proxy-example.png" alt="">
	

## Summary

In short **kubectl proxy** is a very important tool for devops for developing and testing cluster resources. When configuring the proxy, we should make sure the API server node has access to the POD network. Reason is kubectl proxy will connect the cluster resources via API node network. So we should make sure to add proper routes on the API node to make this work.


