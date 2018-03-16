---
layout: post
title: Apache Ignite cluster on Kubernetes
date:   2018-03-16 20:44:00
categories: apache-ignite kubernetes
---

In this blog post we will create an [Apache Ignite](https://ignite.apache.org/) cluster with [Kubernetes](https://kubernetes.io/) and interact with it through Ignite's REST API. We will create a cache, put and get records into/from that cache.

I won't get into the details of concepts like [`pods`](https://kubernetes.io/docs/tutorials/kubernetes-basics/explore-intro/), [`nodes`](https://kubernetes.io/docs/tutorials/kubernetes-basics/explore-intro/), [`services`](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) etc. If you are not familier with those you can always refer to official [documentation](https://kubernetes.io/docs/home/) which is really very well written. Let's get started with installing Kubernetes. I will work with a MacBook Pro running macOS High Sierra but it will be easy to follow on other environments.

#### Installing Kubernetes
We will first install [`kubectl`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl/) which is command line tools to interact with kubernetes:
```sh
$ brew install kubectl
```
Then we will install [`minikube`](https://github.com/kubernetes/minikube) which enables us to run kubernetes locally: 
```sh
$ brew cask install minikube
```
Now we can start our local Kubernetes cluster via running:
```sh
$ minikube start
Starting local Kubernetes v1.9.0 cluster...
Starting VM...
Getting VM IP address...
Moving files into cluster...
Setting up certs...
Connecting to cluster...
Setting up kubeconfig...
Starting cluster components...
Kubectl is now configured to use the cluster.
Loading cached images from config file.
```
Let's check if everything is OK:
```sh
$ kubectl cluster-info
Kubernetes master is running at https://192.168.99.100:8443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
Great! Now our Kubernetes cluster is up and running. You can run:
```sh
$ minikube dashboard
```
to see the [`Kubernetes Dashboard`](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) which is a web-based Kubernetes user interface and a great way to monitor and interact with your cluster.
#### Deploying Apache Ignite nodes
The next step is to deploy Apache Ignite instances to our cluster. For this I created a yaml configuration file as follows: 
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ignite
spec:
  ports:
  - port: 8080
  selector:
    app: ignite
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ignite
spec:
  selector:
    matchLabels:
      app: ignite
  replicas: 4
  template:
    metadata:
      labels:
        app: ignite
    spec:
      containers:
      - name: ignite
        image: alpert/ignite
        ports:
        - containerPort: 8080
        env:
        - name: CONFIG_URI
          value: https://raw.githubusercontent.com/apache/ignite/master/examples/config/example-cache.xml
        - name: OPTION_LIBS
          value: ignite-rest-http
```
Gist link is here: [https://gist.github.com/alpert/9c937ef8806280e964249050648829e2](https://gist.github.com/alpert/9c937ef8806280e964249050648829e2)
The configuration defines a service and a [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/). Service is a NodePort service which enables us to access 8080 port of Kubernetes node. For deployment I didn't use official Apache Ignite docker image because it does not expose port 8080 which is the rest port we will use. Other than that the image is same with the offical one. We use default example cache configuration and enabled the rest module to be able to use it.
So create a file `ignite.yaml` (or whatever you like) with the above content. Then run the following:
```sh
$ kubectl create -f ignite.yaml
service "ignite" created
deployment "ignite" created
```
That creates the deployment and the service which we defined in our `ignite.yaml` file. To see the details of our deployment we can run: 
```sh
$ kubectl describe deployment ignite
Name:                   ignite
Namespace:              default
CreationTimestamp:      Fri, 16 Mar 2018 00:43:03 +0300
Labels:                 app=ignite
Annotations:            deployment.kubernetes.io/revision=1
Selector:               app=ignite
Replicas:               4 desired | 4 updated | 4 total | 4 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=ignite
  Containers:
   ignite:
    Image:  alpert/ignite
    Port:   8080/TCP
    Environment:
      CONFIG_URI:   https://raw.githubusercontent.com/apache/ignite/master/examples/config/example-cache.xml
      OPTION_LIBS:  ignite-rest-http
    Mounts:         <none>
  Volumes:          <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   ignite-66bb989784 (4/4 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  4s    deployment-controller  Scaled up replica set ignite-66bb989784 to 4
```
As you can see we have 4 replications of our application. Also you can see pod details by running: 
```sh
$ kubectl describe pods ignite
Name:           ignite-66bb989784-68qb5
Namespace:      default
Node:           minikube/192.168.99.100
Start Time:     Fri, 16 Mar 2018 00:43:03 +0300
Labels:         app=ignite
                pod-template-hash=2266545340
Annotations:    <none>
Status:         Running
IP:             172.17.0.4
Controlled By:  ReplicaSet/ignite-66bb989784
Containers:
  ignite:
    Container ID:   docker://907cdf40c560f01a7bc7501dc0e06c63e56d295b32f4de0c1ee402a7775a7b09
    Image:          alpert/ignite
    Image ID:       docker://sha256:1d00e63317f649c592686c063cb8626821278b141327b3752790b672c6e1ad22
    Port:           8080/TCP
    State:          Running
      Started:      Fri, 16 Mar 2018 19:11:27 +0300
    Last State:     Terminated
      Reason:       Error
      Exit Code:    137
      Started:      Fri, 16 Mar 2018 00:43:04 +0300
      Finished:     Fri, 16 Mar 2018 09:36:11 +0300
    Ready:          True
    Restart Count:  1
    Environment:
      CONFIG_URI:   https://raw.githubusercontent.com/apache/ignite/master/examples/config/example-cache.xml
      OPTION_LIBS:  ignite-rest-http
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-zrsd5 (ro)
      ............................
```
which has a very long and detailed output so I skip it. You can also describe service details as:
```sh 
$ kubectl describe service ignite
Name:                     ignite
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=ignite
Type:                     NodePort
IP:                       10.107.88.245
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  32036/TCP
Endpoints:                172.17.0.4:8080,172.17.0.5:8080,172.17.0.6:8080 + 1 more...
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```
You can also see all these and much more information on the dashboard.
Everything seems OK on Kubernetes side, now we can check if our Ignite cluster is operating well. To see the logs first we will get the names of pods:
```sh
$ kubectl get pods
NAME                      READY     STATUS    RESTARTS   AGE
ignite-66bb989784-68qb5   1/1       Running   1          18h
ignite-66bb989784-7cqpv   1/1       Running   1          18h
ignite-66bb989784-8z4v5   1/1       Running   1          18h
ignite-66bb989784-sw7hk   1/1       Running   1          18h
```
And then see the logs: 
```sh
$ kubectl logs ignite-66bb989784-sw7hk
[16:11:51]    __________  ________________
[16:11:51]   /  _/ ___/ |/ /  _/_  __/ __/
[16:11:51]  _/ // (7 7    // /  / / / _/
[16:11:51] /___/\___/_/|_/___/ /_/ /___/
[16:11:51]
[16:11:51] ver. 2.4.0#20180305-sha1:aa342270
[16:11:51] 2018 Copyright(C) Apache Software Foundation
[16:11:51]
[16:11:51] Ignite documentation: http://ignite.apache.org
[16:11:51]
[16:11:51] Quiet mode.
[16:11:51]   ^-- Logging to file '/opt/ignite/apache-ignite-fabric-2.4.0-bin/work/log/ignite-d3d4519a.0.log'
[16:11:51]   ^-- Logging by 'JavaLogger [quiet=true, config=null]'
[16:11:51]   ^-- To see **FULL** console log here add -DIGNITE_QUIET=false or "-v" to ignite.{sh|bat}
[16:11:51]
[16:11:51] OS: Linux 4.9.64 amd64
[16:11:51] VM information: OpenJDK Runtime Environment 1.8.0_151-8u151-b12-1~deb9u1-b12 Oracle Corporation OpenJDK 64-Bit Server VM 25.151-b12
[16:11:53] Configured plugins:
[16:11:53]   ^-- None
[16:11:53]
[16:11:53] Message queue limit is set to 0 which may lead to potential OOMEs when running cache operations in FULL_ASYNC or PRIMARY_SYNC modes due to message queues growth on sender and receiver sides.
[16:11:53] Security status [authentication=off, tls/ssl=off]
[16:12:12] Nodes started on local machine require more than 20% of physical RAM what can lead to significant slowdown due to swapping (please decrease JVM heap size, data region size or checkpoint buffer size) [required=1481MB, available=2000MB]
[16:12:20] Performance suggestions for grid  (fix if possible)
[16:12:20] To disable, set -DIGNITE_PERFORMANCE_SUGGESTIONS_DISABLED=true
[16:12:20]   ^-- Enable G1 Garbage Collector (add '-XX:+UseG1GC' to JVM options)
[16:12:20]   ^-- Set max direct memory size if getting 'OOME: Direct buffer memory' (add '-XX:MaxDirectMemorySize=<size>[g|G|m|M|k|K]' to JVM options)
[16:12:20]   ^-- Disable processing of calls to System.gc() (add '-XX:+DisableExplicitGC' to JVM options)
[16:12:20]   ^-- Decrease number of backups (set 'backups' to 0)
[16:12:20] Refer to this page for more performance suggestions: https://apacheignite.readme.io/docs/jvm-and-system-tuning
[16:12:20]
[16:12:20] To start Console Management & Monitoring run ignitevisorcmd.{sh|bat}
[16:12:20]
[16:12:20] Ignite node started OK (id=d3d4519a)
[16:12:20] Topology snapshot [ver=4, servers=4, clients=0, CPUs=8, offheap=1.6GB, heap=4.0GB]
[16:12:20] Data Regions Configured:
[16:12:20]   ^-- default [initSize=256.0 MiB, maxSize=400.0 MiB, persistenceEnabled=false]
```
Everything seems OK here, too. In the logs above there is a line: `[16:11:51]   ^-- Logging to file '/opt/ignite/apache-ignite-fabric-2.4.0-bin/work/log/ignite-d3d4519a.0.log'` which specify the actual log file location. You can also tail it by running:
```sh
$ kubectl exec -it ignite-66bb989784-sw7hk -- tail /opt/ignite/apache-ignite-fabric-2.4.0-bin/work/log/ignite-d3d4519a.0.log
Metrics for local node (to disable set 'metricsLogFrequency' to 0)
    ^-- Node [id=d3d4519a, uptime=00:29:00.388]
    ^-- H/N/C [hosts=4, nodes=4, CPUs=8]
    ^-- CPU [cur=0.33%, avg=1.25%, GC=0%]
    ^-- PageMemory [pages=1232]
    ^-- Heap [used=63MB, free=93.58%, comm=981MB]
    ^-- Non heap [used=54MB, free=96.44%, comm=55MB]
    ^-- Outbound messages queue [size=0]
    ^-- Public thread pool [active=0, idle=0, qSize=0]
    ^-- System thread pool [active=0, idle=6, qSize=0]
```
Here, from the metrics information, you can see that we have 4 nodes in our Ignite cluster: `^-- H/N/C [hosts=4, nodes=4, CPUs=8]`
#### Using Ignite REST API
To access our Ignite cluster, we need the service endpoint. We will get the endpoint for our service as:
```sh
$ minikube service ignite --url
http://192.168.99.100:32036
$ export IGNITE_EP=$(minikube service ignite --url)
```
We will use that endpoint to make rest calls to Ignite. You can check REST API documentation for Ignite from here:[https://apacheignite.readme.io/docs/rest-api](https://apacheignite.readme.io/docs/rest-api). Let's begin with a simple one and get topology information: 
```sh
$ curl $IGNITE_EP/ignite\?cmd\=top
{
   "successStatus":0,
   "error":null,
   "sessionToken":null,
   "response":[
      {
         "nodeId":"8c6d5b27-6195-46f8-8afa-87d7245ee846",
         "consistentId":"127.0.0.1,172.17.0.2:47500",
         "tcpHostNames":[
            "ignite-66bb989784-8z4v5"
         ],
         "tcpPort":11211,
         "metrics":null,
         "caches":[
            {
               "name":"default",
               "mode":"PARTITIONED",
               "sqlSchema":null
            }
         ],
         "order":1,
         "attributes":null,
         "tcpAddresses":[
            "172.17.0.2",
            "127.0.0.1"
         ]
      },
      {
         "nodeId":"17da1e88-27f8-43a2-954d-d345c90e561e",
         "consistentId":"127.0.0.1,172.17.0.7:47500",
         "tcpHostNames":[
            "ignite-66bb989784-7cqpv"
         ],
         "tcpPort":11211,
         "metrics":null,
         "caches":[
            {
               "name":"default",
               "mode":"PARTITIONED",
               "sqlSchema":null
            }
         ],
         "order":2,
         "attributes":null,
         "tcpAddresses":[
            "172.17.0.7",
            "127.0.0.1"
         ]
      },
      {
         "nodeId":"2d578550-2898-4449-8c37-9fc7e2022c5d",
         "consistentId":"127.0.0.1,172.17.0.4:47500",
         "tcpHostNames":[
            "ignite-66bb989784-68qb5"
         ],
         "tcpPort":11211,
         "metrics":null,
         "caches":[
            {
               "name":"default",
               "mode":"PARTITIONED",
               "sqlSchema":null
            }
         ],
         "order":3,
         "attributes":null,
         "tcpAddresses":[
            "172.17.0.4",
            "127.0.0.1"
         ]
      },
      {
         "nodeId":"d3d4519a-8543-45b9-9bc8-71dc896987ad",
         "consistentId":"127.0.0.1,172.17.0.6:47500",
         "tcpHostNames":[
            "ignite-66bb989784-sw7hk"
         ],
         "tcpPort":11211,
         "metrics":null,
         "caches":[
            {
               "name":"default",
               "mode":"PARTITIONED",
               "sqlSchema":null
            }
         ],
         "order":4,
         "attributes":null,
         "tcpAddresses":[
            "127.0.0.1",
            "172.17.0.6"
         ]
      }
   ]
}
```
As you can see we have a default cache named `default`. Lets create another one named `test`:
```sh
$ curl $IGNITE_EP/ignite\?cmd\=getorcreate\&cacheName\=test
{"successStatus":0,"error":null,"sessionToken":null,"response":null}%
```
and put and get a value:
```sh
$ curl $IGNITE_EP/ignite\?cmd\=put\&key\=key\&val\=value\&cacheName\=test
{"successStatus":0,"affinityNodeId":"d3d4519a-8543-45b9-9bc8-71dc896987ad","error":null,"sessionToken":null,"response":true}%
$ curl $IGNITE_EP/ignite\?cmd\=get\&key\=key\&cacheName\=test
{"successStatus":0,"affinityNodeId":"d3d4519a-8543-45b9-9bc8-71dc896987ad","error":null,"sessionToken":null,"response":"value"}%
```
So far so good. From the documentation above you can see all posible REST commands. 
Happy hacking!
