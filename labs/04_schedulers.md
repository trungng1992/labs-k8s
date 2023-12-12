# Schedulers

Labs này tập trung về cơ chế Schedulers của Kubernetes

## Setup Kind cluster với tối thiểu 4 nodes.

```yaml
# tạo file cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cluster-pv
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```

Chạy command `kind` để khởi tạo cluster

```shell
$ kind create cluster --config cluster.yaml
```

```shell
$ kubectl get node
# Kết quả:
NAME                       STATUS   ROLES           AGE     VERSION
cluster-pv-control-plane   Ready    control-plane   6m59s   v1.27.3
cluster-pv-worker          Ready    <none>          6m39s   v1.27.3
cluster-pv-worker2         Ready    <none>          6m39s   v1.27.3
cluster-pv-worker3         Ready    <none>          6m34s   v1.27.3
```

## NodeName

Chỉ định Pod chạy trên một node đặc biệt nào đó.

```yaml
# pod-nodename.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: backend
  name: pod-backend
spec:
  containers:
  - image: ibmcom/guestbook:v1
    imagePullPolicy: IfNotPresent
    name: guestbook
    resources: {}
  nodeName: cluster-pv-worker2 #hostname của node cần chạy
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```
Chạy command sau để tạo pod
```shell
$ kubectl -f pod-nodename.yaml create
# Kiểm tra lại
$ kubectl get pod -o wide

# Kết quả
NAME          READY   STATUS              RESTARTS   AGE   IP       NODE                 NOMINATED NODE   READINESS GATES
pod-backend   0/1     ContainerCreating   0          6s    <none>   cluster-pv-worker2   <none>           <none>
```

## NodeSelectors

NodeSelectors để sử dụng scheduling Pod lên các Nodes có các labels tướng ứng.

Config một labels bất kì cho các nodes cần muốn Pod scheduler lên. Ở labs sẽ chọn 2 node `worker` và `worker3`

```shell
$ kubectl label node cluster-pv-worker app=selector
$ kubectl label node cluster-pv-worker3 app=selector
```

Kiểm tra lại
```shell
$ kubectl get node --show-labels
Kết quả
NAME                       STATUS   ROLES           AGE   VERSION   LABELS
cluster-pv-control-plane   Ready    control-plane   15m   v1.27.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress-ready=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=cluster-pv-control-plane,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
cluster-pv-worker          Ready    <none>          14m   v1.27.3   app=selector,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=cluster-pv-worker,kubernetes.io/os=linux
cluster-pv-worker2         Ready    <none>          14m   v1.27.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=cluster-pv-worker2,kubernetes.io/os=linux
cluster-pv-worker3         Ready    <none>          14m   v1.27.3   app=selector,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=cluster-pv-worker3,kubernetes.io/os=linux
```

Tạo một deployment replicas=2 cho app guestbook

```yaml
# deployment-nodeSelector.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deployment-nodeselector
  name: deployment-nodeselector
spec:
  replicas: 2
  selector:
    matchLabels:
      app: deployment-nodeselector
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: deployment-nodeselector
    spec:
      nodeSelector:
        app: selector
      containers:
      - image: ibmcom/guestbook:v1
        name: guestbook
        resources: {}
```

Kiểm tra

```shell
$ kubectl get pod -o wide | grep deployment

# Kết quả
deployment-nodeselector-594ddbf7fd-2sl2k   1/1     Running   0          14s   10.244.3.5   cluster-pv-worker3   <none>           <none>
deployment-nodeselector-594ddbf7fd-dzck8   1/1     Running   0          11s   10.244.1.7   cluster-pv-worker    <none>           <none>
```

## NodeAffinty

Như nodeSelector tuy nhiên được nhiều điều kiện hơn

```shell
# Delete các labels cũ
$ kubectl label node cluster-pv-worker app=-
$ kubectl label node cluster-pv-worker3 app=-

# thêm các labels khác nhau trên 2 node
$ kubectl label node cluster-pv-worker app=selector1
$ kubectl label node cluster-pv-worker3 app=selector3
```

Tạo file deployment

```yaml
#nodeAffinity1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: deployment-nodeselector
  name: deployment-nodeselector
spec:
  replicas: 2
  selector:
    matchLabels:
      app: deployment-nodeselector
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: deployment-nodeselector
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: app
                operator: In
                values:
                - selector1
      containers:
      - image: ibmcom/guestbook:v1
        name: guestbook
        resources: {}
```

Kiểm tra
```shell
$ kubectl -f nodeAffinity1.yaml create
$ kubectl get pod -o wide

# Kết quả
NAME                                       READY   STATUS    RESTARTS   AGE   IP           NODE                 NOMINATED NODE   READINESS GATES
deployment-nodeselector-7f56776b4c-7hdl4   1/1     Running   0          10s   10.244.1.9   cluster-pv-worker    <none>           <none>
deployment-nodeselector-7f56776b4c-zlm2p   1/1     Running   0          10s   10.244.1.8   cluster-pv-worker    <none>           <none>
```

Tạo file khác
```yaml
# nodeAffinity2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nodeaffinity2
  name: nodeaffinity2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nodeaffinity2
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nodeaffinity2
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: app
                operator: In
                values:
                - selector1
                - selector3
      containers:
      - image: ibmcom/guestbook:v1
        name: guestbook
        resources: {}
```

Kiểm tra
```shell
kubectl -f nodeAffinity2.yaml create
kubectl get pod -o wide | grep nodeaffinity2

# Kết quả
nodeaffinity2-5c6f96bf9d-6jwf6             1/1     Running   0          2s      10.244.1.12   cluster-pv-worker    <none>           <none>
nodeaffinity2-5c6f96bf9d-mpbn8             1/1     Running   0          4s      10.244.3.6    cluster-pv-worker3   <none>           <none>
```

## Pod anti affinity


Tạo file yaml
```yaml
# podantiaffinity.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: podantiaffinity
  name: podantiaffinity
spec:
  replicas: 5
  selector:
    matchLabels:
      app: podantiaffinity
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: podantiaffinity
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - podantiaffinity
            topologyKey: "kubernetes.io/hostname"
      containers:
      - image: ibmcom/guestbook:v1
        name: guestbook
        resources: {}
```

Tạo deployment và kiểm tra
```shell
$ kubectl create -f podantiaffinity.yaml

# Kết quả
NAME                               READY   STATUS    RESTARTS   AGE   IP            NODE                 NOMINATED NODE   READINESS GATES
podantiaffinity-7c64648c7b-hdgpd   1/1     Running   0          2s    10.244.2.6    cluster-pv-worker2   <none>           <none>
podantiaffinity-7c64648c7b-mjtrv   1/1     Running   0          2s    10.244.1.14   cluster-pv-worker    <none>           <none>
podantiaffinity-7c64648c7b-pjm5q   0/1     Pending   0          2s    <none>        <none>               <none>           <none>
podantiaffinity-7c64648c7b-vxfdw   0/1     Pending   0          2s    <none>        <none>               <none>           <none>
podantiaffinity-7c64648c7b-wlvt6   1/1     Running   0          2s    10.244.3.9    cluster-pv-worker3   <none>           <none>
```

## Pod Affinity

```yaml
# pod_affinity.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  selector:
    matchLabels:
      app: web-server
  replicas: 3
  template:
    metadata:
      labels:
        app: web-server
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web-server
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - podantiaffinity
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: web-app
        image: nginx:1.16-alpine
```

Tạo deployment và kết quả
```shell
$ kubectl create -f pod_affinity.yaml

$ kubectl get pod -o wide
# Kết quả
NAME                               READY   STATUS    RESTARTS   AGE     IP            NODE                 NOMINATED NODE   READINESS GATES
podantiaffinity-7c64648c7b-hdgpd   1/1     Running   0          2m45s   10.244.2.6    cluster-pv-worker2   <none>           <none>
podantiaffinity-7c64648c7b-mjtrv   1/1     Running   0          2m45s   10.244.1.14   cluster-pv-worker    <none>           <none>
podantiaffinity-7c64648c7b-pjm5q   0/1     Pending   0          2m45s   <none>        <none>               <none>           <none>
podantiaffinity-7c64648c7b-vxfdw   0/1     Pending   0          2m45s   <none>        <none>               <none>           <none>
podantiaffinity-7c64648c7b-wlvt6   1/1     Running   0          2m45s   10.244.3.9    cluster-pv-worker3   <none>           <none>
web-server-c7c9dc8c6-gf78r         1/1     Running   0          14s     10.244.2.7    cluster-pv-worker2   <none>           <none>
web-server-c7c9dc8c6-p8shs         1/1     Running   0          14s     10.244.3.10   cluster-pv-worker3   <none>           <none>
web-server-c7c9dc8c6-sgng8         1/1     Running   0          14s     10.244.1.15   cluster-pv-worker    <none>           <none>


$ kubectl scale deployment podantiaffinity --replicas=2

$ kubectl get pod -o wide
# Kết quả
NAME                               READY   STATUS    RESTARTS   AGE     IP            NODE                 NOMINATED NODE   READINESS GATES
podantiaffinity-7c64648c7b-hdgpd   1/1     Running   0          5m37s   10.244.2.6    cluster-pv-worker2   <none>           <none>
podantiaffinity-7c64648c7b-mjtrv   1/1     Running   0          5m37s   10.244.1.14   cluster-pv-worker    <none>           <none>
web-server-c7c9dc8c6-gf78r         1/1     Running   0          3m6s    10.244.2.7    cluster-pv-worker2   <none>           <none>
web-server-c7c9dc8c6-p8shs         1/1     Running   0          3m6s    10.244.3.10   cluster-pv-worker3   <none>           <none>
web-server-c7c9dc8c6-sgng8         1/1     Running   0          3m6s    10.244.1.15   cluster-pv-worker    <none>           <none>

$ kubectl rollout restart deployment web-server
$ kubectl get pod -o wide
# Kết quả
NAME                               READY   STATUS    RESTARTS   AGE     IP            NODE                 NOMINATED NODE   READINESS GATES
podantiaffinity-7c64648c7b-hdgpd   1/1     Running   0          6m36s   10.244.2.6    cluster-pv-worker2   <none>           <none>
podantiaffinity-7c64648c7b-mjtrv   1/1     Running   0          6m36s   10.244.1.14   cluster-pv-worker    <none>           <none>
web-server-5bb8f5664-rdhvq         0/1     Pending   0          15s     <none>        <none>               <none>           <none>
web-server-c7c9dc8c6-gf78r         1/1     Running   0          4m5s    10.244.2.7    cluster-pv-worker2   <none>           <none>
web-server-c7c9dc8c6-p8shs         1/1     Running   0          4m5s    10.244.3.10   cluster-pv-worker3   <none>           <none>
web-server-c7c9dc8c6-sgng8         1/1     Running   0          4m5s    10.244.1.15   cluster-pv-worker    <none>           <none>

# Xóa deployment web-server và tạo lại
$ kubectl delete deployment web-server 
$ kubectl create -f pod_affinity.yaml
$ kubectl get pod -o wide

# Kết quả
NAME                               READY   STATUS    RESTARTS   AGE     IP            NODE                 NOMINATED NODE   READINESS GATES
podantiaffinity-7c64648c7b-hdgpd   1/1     Running   0          7m48s   10.244.2.6    cluster-pv-worker2   <none>           <none>
podantiaffinity-7c64648c7b-mjtrv   1/1     Running   0          7m48s   10.244.1.14   cluster-pv-worker    <none>           <none>
web-server-c7c9dc8c6-cn49f         1/1     Running   0          3s      10.244.1.16   cluster-pv-worker    <none>           <none>
web-server-c7c9dc8c6-q8k8g         1/1     Running   0          3s      10.244.2.8    cluster-pv-worker2   <none>           <none>
web-server-c7c9dc8c6-v2rn8         0/1     Pending   0          3s      <none>        <none>               <none>           <none>
```

## Taints and Tolerations

```shell
$ kubectl taint nodes cluster-pv-worker2 app=guestbook:NoSchedule
$ kubectl create deployment guestbook --image=ibmcom/guestbook:v1 --replicas=3
$ kubectl get pod -o wide
# Kết quả
NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE                 NOMINATED NODE   READINESS GATES
guestbook-9cc4d9dcc-7tvrn   1/1     Running   0          17s   10.244.3.12   cluster-pv-worker3   <none>           <none>
guestbook-9cc4d9dcc-8556z   1/1     Running   0          17s   10.244.1.17   cluster-pv-worker    <none>           <none>
guestbook-9cc4d9dcc-gc44z   1/1     Running   0          17s   10.244.3.11   cluster-pv-worker3   <none>           <none>

$ kubectl patch deployment guestbook -p \
'{"spec": {"template": {"spec": {"tolerations": [
{"key": "app", "operator": "Equal", "value": "guestbook", "effect": "NoSchedule"}]}}}}'
$ kubectl get pod -o wide
# Kết quả
NAME                        READY   STATUS    RESTARTS   AGE   IP            NODE                 NOMINATED NODE   READINESS GATES
guestbook-65f7b7976-7m6fd   1/1     Running   0          16s   10.244.2.9    cluster-pv-worker2   <none>           <none>
guestbook-65f7b7976-fclzk   1/1     Running   0          12s   10.244.1.18   cluster-pv-worker    <none>           <none>
guestbook-65f7b7976-rr8lr   1/1     Running   0          14s   10.244.3.13   cluster-pv-worker3   <none>           <none>
```
