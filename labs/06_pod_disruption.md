# Pod Disruption

Mặc định, kiến ​​trúc của `Kubernetes` được thiết kế để luôn đảm bảo tính sẵn có cao mọi lúc, nhưng vẫn có khả năng ứng dụng `container` có thể gặp thời gian chết (downtime) nếu một node gặp sự cố hoặc chúng ta có ý định nâng cấp hệ thống cụm chính mà có thể yêu cầu thay thế các node. Khi các `pods` được triển khai, chúng không nhất thiết phải phân tán đều trên tất cả các `node` trong cụm.

`Schedulers` của Kubernetes sẽ đặt các pods dựa trên tài nguyên có sẵn trên các node, ví dụ nếu chúng ta có 4 pods triển khai cho ứng dụng của chúng ta trên 4 nodes, nếu một trong những worker-node như node-4 bận rộn và chiếm đầy, thì lập lịch có thể xếp 2 pods trên worker-node-1, 1 trên worker-node-2 và 1 trên worker-node-3. Do đó, không có gì được xếp lịch trên worker node 4. Tình hình này trở thành một vấn đề nếu chúng ta đang cố gắng làm trống các node trong quá trình nâng cấp hoặc loại bỏ các node thừa mà có thể đã chạy một số application pods. Vì vậy, nếu một trong những node có pod đang chạy bị loại bỏ, điều này có thể gây ra thời gian chết

## What is Pod Disruption (PBD) ?

Pod Disruption Budget (PDB) cho phép chúng ta xử lý những sự cố ngẫu nhiên đối của các `nodes` bằng cách đảm bảo số lượng `replicas` cần thiết của Pods.

Tạo ra một file manifest sử dụng podDisruption
```yaml
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: test-pdb
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: test-pdb
```

Trong đó 
- `selector.matchLabels.app`: values của labels của deployment muốn kiểm soát.
- `minAvailable`: số lượng tối thiểu của Pods

## Labs

Khởi tạo cluster với Kind.
```yaml
# cluster_pod_disruption.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: pod-disruption
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
  version: v1beta2
  kind: ClusterConfiguration
  patch: |
    - op: add
      path: /apiServer/certSANs/-
      value: my-hostname
# 1 control plane node and 3 workers
nodes:
# the control plane node config
- role: control-plane
- role: worker
- role: worker
- role: worker
```
Chạm command sau để khởi tạo cluster
```shell
kind create cluster --config cluster_pod_disruption.yaml
```

Kiểm tra lại thông tin cluster đã khởi tạo
```shell
kubectl get nodes
```

Tạo manifest deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mynode-app
  namespace: nodejs
spec:
  replicas: 4
  selector:
    matchLabels:
      app: mynode-app
  template:
    metadata:
      labels:
        app: mynode-app
    spec:
      containers:
      - name: mynode-app
        image: matharoo89/mynode-app:08312020
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "64Mi"
            cpu: "150m"
          limits:
            memory: "128Mi"
            cpu: "350m"
```

Chạy command sau để tạo deployment
```shell
$ kubectl -f deployment.yaml create
$ kubectl get pod -n nodejs
NAME                          READY   STATUS    RESTARTS   AGE   IP           NODE                     NOMINATED NODE   READINESS GATES
mynode-app-88dbff789-9zpxz   1/1     Running   0          2m20s   10.244.2.4   pod-disruption-worker3   <none>           <none>
mynode-app-88dbff789-fn7hb   1/1     Running   0          2m19s   10.244.3.6   pod-disruption-worker    <none>           <none>
mynode-app-88dbff789-hgnnn   1/1     Running   0          2m20s   10.244.3.5   pod-disruption-worker    <none>           <none>
mynode-app-88dbff789-kpbj6   1/1     Running   0          2m19s   10.244.1.5   pod-disruption-worker2   <none>           <none>
```

Tạo manifest PBD
```yaml
# pod_disruption.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: mynode-pdb
  namespace: nodejs
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: mynode-app
```

Sử dụng `kubectl` để khởi tạo resoure PodDisruptionBudget
```shell
kubectl -f pod_disruption.yaml create
```

Nhìn vào kết quả trên ta sẽ thấy có 2 pod đang chạy trên 1 node là worker.

Tiến hành drain 1 node (drain là tạm thời remove node đó ra khởi cluster)

```shell
$ kubectl drain pod-disruption-worker --ignore-daemonsets
node/pod-disruption-worker cordoned
Warning: ignoring DaemonSet-managed Pods: kube-system/kindnet-vlgwm, kube-system/kube-proxy-b88vd
evicting pod nodejs/mynode-app-88dbff789-hgnnn
evicting pod nodejs/mynode-app-88dbff789-fn7hb
error when evicting pods/"mynode-app-88dbff789-hgnnn" -n "nodejs" (will retry after 5s): Cannot evict pod as it would violate the pod's disruption budget.
evicting pod nodejs/mynode-app-88dbff789-hgnnn
pod/mynode-app-88dbff789-fn7hb evicted
pod/mynode-app-88dbff789-hgnnn evicted
node/pod-disruption-worker drained
```

Khi cố gắng drain node `pod-disruption-worker` ta sẽ thấy lỗi `error when evicting pods/"mynode-app-88dbff789-hgnnn" -n "nodejs" (will retry after 5s): Cannot evict pod as it would violate the pod's disruption budget.`. PBD sẽ không cho phép một node bị removed tới khi số lượng tối thiểu replica của Deployment có thể available trở lại.

```shell
$ kubectl -n nodejs get pod -o wide
NAME                         READY   STATUS    RESTARTS   AGE     IP           NODE                     NOMINATED NODE   READINESS GATES
mynode-app-88dbff789-9zpxz   1/1     Running   0          4m30s   10.244.2.4   pod-disruption-worker3   <none>           <none>
mynode-app-88dbff789-kpbj6   1/1     Running   0          4m29s   10.244.1.5   pod-disruption-worker2   <none>           <none>
mynode-app-88dbff789-zbdmb   1/1     Running   0          41s     10.244.2.5   pod-disruption-worker3   <none>           <none>
mynode-app-88dbff789-zxsjv   1/1     Running   0          47s     10.244.1.6   pod-disruption-worker2   <none>           <none>
```