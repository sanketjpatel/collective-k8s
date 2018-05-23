# Collective Talk - Introduction to Kubernetes

This repository contains various commands, manifests, and docs for the Collective talk on 05/24/2018.

Try the following commands on your kubernetes cluster:

```bash
$ kubectl create namespace collective
$ kubectl config set-context $(kubectl config current-context) --namespace=collective
$ kubectl apply -f 01-kuard-pod.yaml
$ kubectl get pods
$ kubectl describe pods
$ kubectl apply -f 02-kuard-pod-health.yaml
```

