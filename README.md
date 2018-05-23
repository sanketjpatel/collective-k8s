# Collective Talk - Introduction to Kubernetes

This repository contains various commands, manifests, and docs for the Collective talk on 05/24/2018. A lot of the following material is inspired and borrowed from the [Kubernetes: Up and Running](http://shop.oreilly.com/product/0636920043874.do) book.

## Setup

Try the following commands on your kubernetes cluster:

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
selector:
  matchLabels:
    app: alpaca
  matchExpressions:
    - {key: ver, operator: In, values: [1, 2]}
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

