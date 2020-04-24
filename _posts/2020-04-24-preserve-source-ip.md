---
layout: post
title: Preserve Source IP with NodePort and HA Proxy
comments_id: 5
---
{% seo %}



Below is the architecture of the deployment. In this deployment we have used Nginx ingress controller  for proxying traffic to backend services. Nginx ingress is exposed via NodePort service and external HA proxy is used to receive external traffic.

<img src="/assets/images/nginx-ingress-with-ssl.png" alt="">

When we checked the Nginx Ingress controller logs we could see the worker node replaces the source IP address (SNAT) in the packet with its own IP address.


```sh
$ kubectl   logs    ingress-controller-57f4585dfd-qhspr   -f -n nginx-ingress

10.44.0.0 - [10.44.0.0] - - [11/Apr/2020:20:01:53 +0000] "GET /app1/sample/hello.jsp HTTP/2.0" 200 233 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.113 Safari/537.36" 370 0.007 [default-tomcat-service-app1-80] 10.44.0.4:8080 355 0.008 200 0237cc8b5adc0fab8a3063295d583158
```

**10.44.0.0** is the IP of the one of the worker nodes from the POD network range.

According to the official [doc](https://kubernetes.io/docs/tutorials/services/source-ip/#source-ip-for-services-with-type-clusterip) we could see by default traffic sent to the NodePort services are source NAT’d. So using the **externalTrafficPolicy:Local** we can change this behaviour to preserve the source IP.

```sh
$ kubectl explain Service.spec.externalTrafficPolicy
KIND:     Service
VERSION:  v1

FIELD:    externalTrafficPolicy <string>

DESCRIPTION:
     externalTrafficPolicy denotes if this Service desires to route external
     traffic to node-local or cluster-wide endpoints. "Local" preserves the
     client source IP and avoids a second hop for LoadBalancer and Nodeport type
     services, but risks potentially imbalanced traffic spreading. "Cluster"
     obscures the client source IP and may cause a second hop to another node,
     but should have good overall load-spreading.
```

Below is the clear diagram which shows the difference among two options for **externalTrafficPolicy**.

<img src="/assets/images/externaltrafficpolicy.png" alt="">
###### Image by <a href="https://www.asykim.com/blog/deep-dive-into-kubernetes-external-traffic-policies"> asykim</a>

There are pros and cons of both policy types and I am not going to discuss it here you can find those in this [post](https://www.asykim.com/blog/deep-dive-into-kubernetes-external-traffic-policies).
In my setup I am using **weave** as network provider, so in order to above policy to work I have to enable **NO_MASQ_LOCAL** on weave network pods according to [this](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/). 


*NO_MASQ_LOCAL - set to 1 to preserve the client source IP address when accessing Service annotated with service.spec.externalTrafficPolicy=Local. The feature works only with Weave IPAM (default)*

Now let’s enable this on weave pods. 
```sh
$ kubectl patch daemonsets.apps -n kube-system weave-net --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/env/-", "value": {"name": "NO_MASQ_LOCAL", "value": "1" } }]'
```
```sh
$ kubectl  describe daemonset weave-net -n kube-system
…
Environment:
      HOSTNAME:        (v1:spec.nodeName)
      NO_MASQ_LOCAL:  1
...
```
We can see the new config is added to the weave pods.

Next we will enable the policy on the NodePort service which Nginx Ingress is exposed to.
```sh
$ kubectl patch svc ingress -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```sh
$kubectl  describe svc ingress  -n nginx-ingress

Name:                     ingress
Namespace:                nginx-ingress
Labels:                   <none>
Annotations:              <none>
Selector:                 name=nginx-ingress
Type:                     NodePort
IP:                       10.96.0.62
Port:                     http  80/TCP
TargetPort:               80/TCP
NodePort:                 http  30111/TCP
Endpoints:                10.32.0.3:80
Port:                     https  443/TCP
TargetPort:               443/TCP
NodePort:                 https  30112/TCP
Endpoints:                10.32.0.3:443
Session Affinity:         None
External Traffic Policy:  Local
Events:                   <none>

```
We can see **External Traffic Policy:  Local** is enabled now.

Now if we directly send traffic to Nodeport service , client IP should be preserved. But in my setup I am using HA proxy in TCP mode. So in order to make this work I have to do a few more things.

In HA proxy we need to send client IP details to the NodePort service. For this we add **send-proxy** as below in backend service definition.

```sh

listen stats 192.168.1.100:8080
    mode http
    log global
    stats enable
    stats realm Haproxy\ Statistics
    stats uri /haproxy_stats
    stats hide-version
    stats auth admin:admin@rserve

frontend rserve_frontend
    bind *:443
    mode tcp
    option tcplog
    timeout client  1m
    default_backend rserve_backend

backend rserve_backend
    mode tcp
    option tcplog
    option log-health-checks
    option redispatch
    log global
    balance roundrobin
    timeout connect 10s
    timeout server 1m
    server rserve1 192.168.5.21:30112 send-proxy check
    server rserve1 192.168.5.22:30112 send-proxy check

```
Once we added the  **send-proxy** ,HA proxy uses the [proxy protocol](https://docs.nginx.com/nginx/admin-guide/load-balancer/using-proxy-protocol/) to communicate with backend Nginx controllers. So now we need to enable proxy protocol in the Nginx controller. For this we pass the config options via the config map configured in Nginx controller.

```sh
apiVersion: v1
data:
  compute-full-forwarded-for: "true"
  use-forwarded-headers: "true"
  use-proxy-protocol: "true"
kind: ConfigMap
metadata:
  creationTimestamp: "2020-04-05T06:51:25Z"
  name: nginx-config
  namespace: nginx-ingress

```
I have added the below section to the above config map (nginx-config)  according to the [doc](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#compute-full-forwarded-for) to accept proxy protocol and preserve source IP for upstreams as well.
```sh
data:
  compute-full-forwarded-for: "true"
  use-forwarded-headers: "true"
  use-proxy-protocol: "true"
```

Now we are done. Let’s see the Nginx Controller Logs again.

```sh
$ kubectl   logs    ingress-controller-57f4585dfd-qhspr   -f -n nginx-ingress
192.168.1.4 - [192.168.1.4] - - [12/Apr/2020:04:11:57 +0000] "GET /app1/sample/hello.jsp HTTP/2.0" 200 233 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.113 Safari/537.36" 36 0.003 [default-tomcat-service-app1-80] 10.32.0.2:8080 355 0.004 200 7651ebffc41710f6371c07261b06aacd
```
We can see now the source IP is preserved.

## Summary
When using **weave** as POD network we must enable **NO_MASQ_LOCAL** option to enable the **service.spec.externalTrafficPolicy=Local** config on NodePort service. Additionally as per the nature of **service.spec.externalTrafficPolicy=Local** policy it is recommended to run at least one pod per node and it says that if the node receives the traffic which pod does not reside traffic will be dropped. But in my setup I did  not experience such a behavior. Finally in HA proxy environment we must add **send-proxy** in HA config and enable proxy protocol in Nginx Ingress controller. Anyway it is a good idea to deploy Nginx Ingress as [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) to ensure at least one pod is runs on every node.

