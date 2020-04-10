---
layout: post
title: Deploy Nginx Load Balancer with SSL on Kubernetes
---
{% seo %} 
## Description

I will be using 2 tomcat pods as two applications  in the backend and expose those via Nginx ingess controller. There are other ingress controller also available. For more information please [refer this link](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).

### Architecture of the deployment

Here is the overview of the deployment. In here I have used ClusterIP type service for exposing tomcat service internally to Ingress Controller. For exposing Nginx ingress controller to outside I have used NodePort type service. Finally NodePort service is fronted by HAProxy server which faces the external traffic. 

<img src="/assets/images/nginx-ingress-with-ssl.png" alt="">



#### Now let's go though the deployment with details
```
Kubernetes version: Kubernetes v1.13.0
Nodes : 192.168.5.21 ,192.168.5.22
HA Proxy : 192.168.1.5
```
 

Initially we create namespace called **"nginx-ingress"** to group NginX controller and it's resources.

**NginX Ingress Controller :** This is specally design Nginx POD which act as reverse proxy and loadbalancer. For this controller to be able to act as cluster wide ingress controller we should grant required privileges. This is accomplished by creating ClusterRoles , ClusterRoleBindings  and service account objects. 

**Service Account :**  Service accounts are created in order to facilitate processes running inside the containers to access the apiserver.

**Cluster Role :** Permissions assigned to a role that apply to an entire cluster

**Cluster Role Bindings :** Binding a ClusterRole to a specific account

***Now we will create above mentioned resources in the cluster***

* Namespace

nginx-ingress-namespace.yaml
```
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-ingress
```
```sh
$ kubectl create -f nginx-ingress-namespace.yaml
```
```sh
$ kubectl  get namespace nginx-ingress
NAME            STATUS   AGE
nginx-ingress   Active   3d14h

```
* Service Account

nginx-ingress-sa.yaml
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress
  namespace: nginx-ingress
```
```sh
$ kubectl create -f nginx-ingress-sa.yaml
```
```sh
$ kubectl  get sa -n nginx-ingress
NAME            SECRETS   AGE
default         1         3d14h
nginx-ingress   1         3d14h
```
* Cluster Role

nginx-ingress-cr.yaml

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  creationTimestamp: "2020-04-08T10:41:18Z"
  name: nginx-ingress
  resourceVersion: "153232"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterroles/nginx-ingress
  uid: 7adafc73-7985-11ea-a7bd-02854f0befe7
rules:
- apiGroups:
  - ""
  resources:
  - services
  - endpoints
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  verbs:
  - get
  - list
  - watch
  - update
  - create
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  verbs:
  - list
  - watch
  - get
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs:
  - list
  - watch
  - get
- apiGroups:
  - extensions
  resources:
  - ingresses/status
  verbs:
  - update
- apiGroups:
  - k8s.nginx.org
  resources:
  - virtualservers
  - virtualserverroutes
  - globalconfigurations
  - transportservers
  verbs:
  - list
  - watch
  - get
```

```sh
$ kubectl create -f nginx-ingress-cr.yaml
```
```sh
$ kubectl  get clusterroles nginx-ingress
NAME            AGE
nginx-ingress   10h
```
* Cluster Role Bindings

nginx-ingress-crb.yaml

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: nginx-ingress
subjects:
- kind: ServiceAccount
  name: nginx-ingress
  namespace: nginx-ingress
roleRef:
  kind: ClusterRole
  name: nginx-ingress
  apiGroup: rbac.authorization.k8s.io

```

```sh
$ kubectl create -f nginx-ingress-crb.yaml
```
```sh
$ kubectl  get clusterrolebindings nginx-ingress
NAME            AGE
nginx-ingress   10h
```

* Adding  config map which can used to pass any variables later to the Nginx controller.
```
--configmap=$(POD_NAMESPACE)/nginx-config
```
Creating empty configmap for now.

```sh
$ kubectl create configmap nginx-config -n nginx-ingress
```


* Let's deploy NginX Ingress Controller now

nginx-ingress-controller.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-controller
  namespace: nginx-ingress
spec:
  replicas: 1
  selector:
    matchLabels:
      name: nginx-ingress
  template:
    metadata:
      labels:
        name: nginx-ingress
    spec:
      serviceAccountName: nginx-ingress
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.21.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-config
            - --default-backend-service=nginx-ingress/default-http-backend
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443

```

```sh
$ kubectl create -f nginx-ingress-controller.yaml
```
```sh
$ kubectl  get deployments -n nginx-ingress
NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
ingress-controller   1/1     1            1           3d12h
```

* Adding a NodePort service to expose the Nginx ingress controller to the outside of the cluster. In here we expose 30111(redirect traffic to  port 80 in the nginx controller ) and 30112 (redirect traffic to  port 443 in the nginx controller), reason for using higher ports is only ports between 30000-32767  are allowed to used for this purpose.

nginx-ingress-expose.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: ingress
  namespace: nginx-ingress
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    nodePort: 30111
    name: http
  - port: 443
    targetPort: 443
    nodePort: 30112
    protocol: TCP
    name: https
  selector:
    name: nginx-ingress

```

```sh
$ kubectl create -f nginx-ingress-expose.yaml
```

```sh
$ kubectl  get svc -n nginx-ingress

NAME                   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
default-http-backend   ClusterIP   10.96.0.141   <none>        80/TCP                       3d12h
ingress                NodePort    10.96.0.213   <none>        80:30111/TCP,443:30112/TCP   3d12h


Here you can see default-http-backend  service, this is configured in Nginx controller as default backend (- --default-backend-service=nginx-ingress/default-http-backend)  which send all non matching request. For this we can simply deploy a pod and create a ClusterIP type service to send non matching traffic.

```
* Finally we are adding HA proxy to properly expose the NodePort service to outside using standard ports ( 80 and 443). Further in the HA proxy we send traffic to both nodes in the cluster.

/etc/haproxy/haproxy.cfg

```
listen stats 192.168.1.5:8080
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
    server rserve1 192.168.5.21:30112 check
    server rserve1 192.168.5.22:30112 check
```
#### At this point we have completed Ingress controller and it's component deployment. Next step is to deploy applications and it's configurations.

* Now let's create Ingress resources, prior to that I have deployed 2 tomcat pods with sample applictions which will be exposed to outside.

*Creating tomcat POD*
```sh
$ kubectl run  tomcat-app1 --image tomcat
$ kubectl run  tomcat-app2 --image tomcat
```
*Since this is a testing I am going to add sample WAR files to each pods temporarily*
```sh
$ kubectl  exec -it tomcat-app1-*** -- /bin/bash
$ cd webapps; wget "https://github.com/AKSarav/SampleWebApp/raw/master/dist/SampleWebApp.war"

$ kubectl  exec -it tomcat-app2-*** -- /bin/bash
$ cd webapps; wget "https://tomcat.apache.org/tomcat-7.0-doc/appdev/sample/sample.war"
```

* Now we will create services to expose tomcat pods.

tomcat-service-app1.yaml
```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2020-04-05T05:00:53Z"
  name: tomcat-service-app1
  namespace: default
  resourceVersion: "84908"
  selfLink: /api/v1/namespaces/default/services/tomcat-service
  uid: 6d307883-76fa-11ea-bba8-02854f0befe7
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    run: tomcat-app1
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```
```sh

$ kubectl create -f tomcat-service-app1.yaml

```

tomcat-service-app2.yaml
```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2020-04-05T05:00:53Z"
  name: tomcat-service-app2
  namespace: default
  resourceVersion: "84908"
  selfLink: /api/v1/namespaces/default/services/tomcat-service
  uid: 6d307883-76fa-11ea-bba8-02854f0befe7
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    run: tomcat-app2
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```
```sh

$ kubectl create -f tomcat-service-app2.yaml

```

* Final part of this deployment is to create Ingress resource and secret to store the certificates

We have generated required SSL certificates form Let's Encrypt.We will be using generated certificates to create the secret object.

```sh
$ kubectl create secret tls tomcat-app.chakraya.com-certs   --cert tomcat-app.chakraya.com.crt  --key tomcat-app.chakraya.com.key

```

tomcat-ingress-resource.yaml
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rewrite-target: /
  creationTimestamp: "2020-04-08T12:23:39Z"
  generation: 1
  name: tomcat-ingress-resource
  namespace: default
  resourceVersion: "162735"
  selfLink: /apis/extensions/v1beta1/namespaces/default/ingresses/tomcat-ingress-resource
  uid: c6f81dab-7993-11ea-a7bd-02854f0befe7
spec:
  rules:
  - host: tomcatapp.chakraya.com
    http:
      paths:
      - backend:
          serviceName: tomcat-service-app1
          servicePort: 80
        path: /app1
      - backend:
          serviceName: tomcat-service-app2
          servicePort: 80
        path: /app2
  tls:
  - hosts:
    - tomcatapp.chakraya.com
    secretName: tomcat-app.chakraya.com-certs
status:
  loadBalancer:
    ingress:
    - {}
```

```sh
$ kubectl  create -f tomcat-ingress-resource.yaml
```

When we add Ingress resource we can see how NginX ingress controller reload it's configuration.

```sh
kubectl   logs    ingress-controller-***   -f -n nginx-ingress
```

```
I0408 10:43:26.411953       7 backend_ssl.go:60] Updating Secret "default/tomcat-app.chakraya.com-certs" in the local store
W0408 10:43:26.412206       7 controller.go:373] Service "nginx-ingress/default-http-backend" does not have any active Endpoint
I0408 10:43:26.412305       7 controller.go:172] Configuration changes detected, backend reload required.
I0408 10:43:26.413193       7 event.go:221] Event(v1.ObjectReference{Kind:"Ingress", Namespace:"default", Name:"tomcat-ingress-resource", UID:"c5b3a5cf-7985-11ea-a7bd-02854f0befe7", APIVersion:"extensions/v1beta1", ResourceVersion:"153426", FieldPath:""}): type: 'Normal' reason: 'CREATE' Ingress default/tomcat-ingress-resource
I0408 10:43:26.567308       7 controller.go:190] Backend successfully reloaded.
[08/Apr/2020:10:43:26 +0000]TCP200000.000
I0408 10:43:26.803402       7 status.go:388] updating Ingress default/tomcat-ingress-resource status to [{ }]
I0408 10:43:26.808579       7 event.go:221] Event(v1.ObjectReference{Kind:"Ingress", Namespace:"default", Name:"tomcat-ingress-resource", UID:"c5b3a5cf-7985-11ea-a7bd-02854f0befe7", APIVersion:"extensions/v1beta1", ResourceVersion:"153429", FieldPath:""}): type: 'Normal' reason: 'UPDATE' Ingress default/tomcat-ingress-resource
W0408 10:43:29.746457       7 controller.go:373] Service "nginx-ingress/default-http-backend" does not have any active Endpoint
I0408 10:43:51.474229       7 backend_ssl.go:189] Updating local copy of SSL certificate "default/tomcat-app.chakraya.com-certs" with missing intermediate CA certs
W0408 10:43:51.474385       7 controller.go:373] Service "nginx-ingress/default-http-backend" does not have any active Endpoint
```
All done. We can now access the backend as below.

*Accessing app1*
<img src="/assets/images/nginx-ingress-with-ssl-tomcatapp1.png" alt="">

*Accessing app2*
<img src="/assets/images/nginx-ingress-with-ssl-tomcatapp2.png" alt="">

*ssl certificate created via Ingress resource*
<img src="/assets/images/nginx-ingress-with-ssl-cert.png" alt="">

## Summary 

In this example we have  used Kubernetes for hosting SSL based application fronted by HA proxy. Reason for using HA proxy is to allow outside traffic on  standard SSL port (443) instead of higher ports exposed by NodePort service. Also to load balance traffic over two cluster nodes.  
Ingress controller and it's componenets are one time  deployement.  So adding extra Virtual host can be achieved only by creating another Ingress resource as appropriate.
