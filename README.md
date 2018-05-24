# Collective Talk - Introduction to Kubernetes

This repository contains various commands, manifests, and docs for the Collective talk on 05/24/2018. A lot of the following material is inspired and borrowed from the [Kubernetes: Up and Running](http://shop.oreilly.com/product/0636920043874.do) book.

## Setup

Try the following commands on your Kubernetes cluster:

Create a namespace for your collective playground and set context

```bash
$ kubectl create namespace collective
$ kubectl config set-context $(kubectl config current-context) --namespace=collective
```

## Working with Pods

Create a kuard pod

```bash
$ kubectl apply -f 01-kuard-pod.yaml
```

See all the pods in this namespace using `get`, read more details and description of the object using `describe`, `log` to get object's logs.

```bash
$ kubectl get pods
$ kubectl describe pods
$ kubectl get logs pods
```

Delete the object using

```bash
$ kubectl delete pods kuard
```

Add health checks

```bash
$ kubectl apply -f 02-kuard-pod-health.yaml
```

Add resource requests and limits

```bash
$ kubectl apply -f 03-kuard-pod-resreq.yaml
```

Add a volume to the pod

```bash
$ kubectl apply -f 04-kuard-pod-volume.yaml
```

## Working with Labels and Annotations

As you run more applications on Kubernetes, the resources/objects scale in size and complexity. Labels and annotations let you work in sets of things that map how you think about your application. You can organize, mark, and cross-index resources to represent groups that make the most sense of your application.

### Labels

Labels provide the foundation for grouping objects.
Annotations provide a storage mechanism to hold nonidentifying information (metadata) that can be leveraged by other tools and libraries.

Run a few deployments and add labels to them

```bash
$ kubectl run alpaca-prod \
  --image=gcr.io/kuar-demo/kuard-amd64:1 \
  --replicas=2 \
  --labels="ver=1,app=alpaca,env=prod"

$ kubectl run alpaca-test \
  --image=gcr.io/kuar-demo/kuard-amd64:2 \
  --replicas=1 \
  --labels="ver=2,app=alpaca,env=test"


$ kubectl run bandicoot-prod \
  --image=gcr.io/kuar-demo/kuard-amd64:2 \
  --replicas=2 \
  --labels="ver=2,app=bandicoot,env=prod"

$ kubectl run bandicoot-staging \
  --image=gcr.io/kuar-demo/kuard-amd64:2 \
  --replicas=1 \
  --labels="ver=2,app=bandicoot,env=staging"
```

Check the deployments

```bash
$ kubectl get deployments --show-labels
```

Modify the label for one of the deployments. Labels can be applied/updated after the object is created.

```bash
$ kubectl label deployments alpaca-test "canary=true"
```

Use the -L option to show a label value as a column

```bash
$ kubectl get deployments -L canary
```

Remove a label by applying a `-` suffix

```bash
$ kubectl label deployments alpaca-test "canary-"
```

### Selectors

Selectors are a way to find objects based on their labels.
Check the pods

```bash
$ kubectl get pods --show-labels
```

Show pods on version 2

```bash
$ kubectl get pods --selector="ver=2"
```

Show pods with multiple selectors. (Logical AND)

```bash
$ kubectl get pods --selector="ver=2,app=bandicoot"
```

Show pods with labels matching a set of values. (Logical OR)

```bash
$ kubectl get pods --selector="env in (prod,staging)"
```

Selectors are also used in the YAML manifests to refer to Kubernetes objects. A selector of `app=alpaca,ver in (1,2)` would translate to the following:

```yaml
...
selector:
  matchLabels:
    app: alpaca
  matchExpressions:
    - {key: ver, operator: In, values: [1, 2]}
...
```

### Annotations

Annotations provide a place to store additional metadata for Kubernetes objects with the sole purpose of assisting tools and libraries. While labels are used to identify and group objects, annotations are used to provide extra information about where and object came from, how to use it, or policy around that object. When in doubt, add information to an object as an annotation and promote it to a label if you find yourself wanting to use it in a selector.

They can be defined in the common `metadata` section in every Kubernetes object.

```yaml
...
metadata:
  annotations:
    example.com/icon-url: "https://example.com/icon.png"
...
```

Annotations are convenient and provide powerful loose coupling, but should be used judiciously to avoid an untyped mess of data.

#### Cleanup

Delete the deployments created in this section.

```bash
$ kubectl delete deployments --all
```

## Service Discovery

Service discovery tools help solve the problem of finding which processes are listening at which addresses for which services. A good service discovery system will enable users to resolve this information quickly and reliably. Such a system is low-latency. Kubernetes offers a `Service` object to create a _named label selector_. `kubectl expose` is used to create a service for a deployment.

Let's create a deployment

```bash
$ kubectl run alpaca-prod \
  --image=gcr.io/kuar-demo/kuard-amd64:1 \
  --replicas=3 \
  --port=8080 \
  --labels="ver=1,app=alpaca,env=prod"
```

Expose this deployment by creating a service

```bash
$ kubectl expose deployment alpaca-prod
```

Check the service

```bash
$ kubectl get services -o wide
```

Let's create another deployment

```bash
$ kubectl run bandicoot-prod \
  --image=gcr.io/kuar-demo/kuard-amd64:2 \
  --replicas=2 \
  --port=8080 \
  --labels="ver=2,app=bandicoot,env=prod"
```

Create a service for this deployment

```bash
$ kubectl expose deployment bandicoot-prod
```

Check services

```bash
$ kubectl get services -o wide
```

The `SELECTOR` column indicates that the `alpaca-prod` service just gives a name to a selector, and specifies which ports to talk to for that service. The `kubectl expose` command pulls both the label selector and relevant ports from the deployment definition.

The service is also assigned a new type of virtual IP called a _Cluster IP_. This is a special IP address that the system will load-balance across all of the pods that are identified by the selector.

The cluster IP does not change, so it is appropriate to give it a DNS address. The issues with clients caching DNS results no longer apply.


```bash
$ ALPACA_POD=$(kubectl get pods -l app=alpaca \
  -o jsonpath='{.items[0].metadata.name}')

$ kubectl port-forward $ALPACA_POD 8090:8080
```

If you open the DNS Query section on the kuard app, and query `bandicoot-prod` for DNS Type `A`, you will see the following output.

```bash
;; opcode: QUERY, status: NOERROR, id: 43754
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;bandicoot-prod.collective.svc.cluster.local.	IN	 A

;; ANSWER SECTION:
bandicoot-prod.collective.svc.cluster.local.	30	IN	A	100.64.41.4
```

### Readiness Checks

A readiness check is a way for an overloaded server to signal to the system that it doesn't want to receive traffic anymore. This is a great way to implement graceful shutdown. The server can signal that it no longer wants traffic, wait until existing connections are closed, and then cleanly exit.

Let's add a readiness check to our deployment.

```bash
$ kubectl edit deployment/alpaca-prod
```

Add a readiness check to the pod spec

```yaml
...
spec:
  ...
  template:
    ...
    spec:
      containers:
        ...
        name: alpaca-prod
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          periodSeconds: 2
          initialDelaySeconds: 0
          failureThreshold: 3
          successThreshold: 1
...
```

A pod with failing readiness check is removed from the service loadbalancer, so no more connections will be made to the pod via the service until it is ready.

You can confirm this by watching the endpoints of the service

```bash
$ kubectl get endpoints alpaca-prod --watch
```

Go to the browser and click the `Fail` link in the `Readiness Probe` tab. You should see the endpoint corresponding to the pod removed from the `alpaca-prod` service.

### Service Types

So far, we have covered exposing services inside of a cluster. Oftentimes, the IPs for pods are only reachable from within the cluster. There are a few ways to allow external traffic reach the pods.

#### NodePort

For a service of type _NodePort_, the system picks a port (or user specifies one), and every node in the cluster then forwards traffic from that port to the service. This is in addition to the Cluster IP that's already assigned to the service. With this feature, if you can reach any node in the cluster, you can reach the service too.

```bash
$ kubectl edit service alpaca-prod
```

Modify `.spec.type` from `ClusterIP` to `NodePort`.

```bash
$ kubectl describe svc alpaca-prod

Name:                     alpaca-prod
Namespace:                collective
Labels:                   app=alpaca
                          env=prod
                          ver=1
Annotations:              <none>
Selector:                 app=alpaca,env=prod,ver=1
Type:                     NodePort
IP:                       100.65.8.237
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  31212/TCP
Endpoints:                100.96.5.108:8080,100.96.6.16:8080,100.96.7.93:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

#### LoadBalancer

You can use `LoadBalancer` service type to create a new loadbalancer on your cloud provider and direct it to the nodes in your cluster. Basically, it is a superset of the `NodePort` service. In this case, since I'm running on AWS infrastructure, a classic ELB gets provisioned.

```bash
$ kubectl edit svc alpaca-prod
```

Modify `.spec.type` from `NodePort` to `LoadBalancer`

```bash
$ kubectl describe svc alpaca-prod

Name:                     alpaca-prod
Namespace:                collective
Labels:                   app=alpaca
                          env=prod
                          ver=1
Annotations:              <none>
Selector:                 app=alpaca,env=prod,ver=1
Type:                     LoadBalancer
IP:                       100.65.8.237
LoadBalancer Ingress:     aebad48a95ee111e89b82022fc41c72f-225282263.us-east-1.elb.amazonaws.com
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  31212/TCP
Endpoints:                100.96.5.108:8080,100.96.6.16:8080,100.96.7.93:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason                Age   From                Message
  ----    ------                ----  ----                -------
  Normal  Type                  13s   service-controller  NodePort -> LoadBalancer
  Normal  EnsuringLoadBalancer  13s   service-controller  Ensuring load balancer
  Normal  EnsuredLoadBalancer   10s   service-controller  Ensured load balancer

```

You can grab the `LoadBalancer Ingress` and open up a browser

```bash
$ LB_ING=$(kubectl get service alpaca-prod -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

$ curl http://$LB_ING:8080
```
