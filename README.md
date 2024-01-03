# kustomize_base


## This is an example repo for using kustomize to manage K8s manifests and deploying to ArgoCD.


## Setup a test cluster

Setup a test cluster with `Kind`.

kind create cluster --config cluster.yaml

`cluster.yaml`:
```
# this config file contains all config fields with comments
# NOTE: this is not a particularly useful config file
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
# patch the generated kubeadm config with some extra settings
kubeadmConfigPatches:
  - |
    apiVersion: kubelet.config.k8s.io/v1beta1
    kind: KubeletConfiguration
    evictionHard:
      nodefs.available: "0%"
# patch it further using a JSON 6902 patch
kubeadmConfigPatchesJSON6902:
  - group: kubeadm.k8s.io
    version: v1beta3
    kind: ClusterConfiguration
    patch: |
      - op: add
        path: /apiServer/certSANs/-
        value: my-hostname
# 2 control plane node and 2 workers
nodes:
  # the control plane node config
  - role: control-plane
  - role: control-plane
  # the two workers
  - role: worker
  - role: worker

```

## K8s Config Architecture

This is a single repo but the idea is to treat the `dev` as a directory living in a source code repo and the `base` directory living in a **remote** mono repo of some sort that can be treated as a resuable template for many micro services.

![image](docs/img/kustomize.jpg)


### Example

## Deploy ArgoCD


```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml


kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Get the initial password:

```
argocd admin initial-password -n argocd
```

browse to https://localhost:8080

## Deploy the sample app


Deploy via ArgoCD...the following builds the app definition within ArgoCD and it will then proceed
with syncing the K8s manifests to the cluster.

```
kubectl apply -f kustomize/backend-api/dev/app.yaml
```

To see the manifest that `kustomize` will generate:
```
kustomize build kustomize/backend-api/dev

apiVersion: v1
kind: Namespace
metadata:
  labels:
    app: backend-api
  name: backend-api-dev
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: backend-api
  name: backend-api
  namespace: backend-api-dev
spec:
  ports:
  - name: http
    port: 8080
  selector:
    app: backend-api
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: backend-api
  name: backend-api
  namespace: backend-api-dev
spec:
  selector:
    matchLabels:
      app: backend-api
  template:
    metadata:
      labels:
        app: backend-api
    spec:
      containers:
      - image: nginx:1.15.0
        name: backend-api
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        resources:
          limits:
            cpu: ".5"
            memory: 128Mi
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  labels:
    app: backend-api
  name: backend-api
  namespace: backend-api-dev
spec:
  maxReplicas: 10
  metrics:
  - resource:
      name: cpu
      target:
        averageUtilization: 70
        type: Utilization
    type: Resource
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend-api
```
