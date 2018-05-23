# Collective Talk - Introduction to Kubernetes

This repository contains various commands, manifests, and docs for the Collective talk on 05/24/2018.

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



