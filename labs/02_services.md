# Services 

Yêu cầu đã làm thuần deployment

Để expose một services ta sử dụng command như sau:

```shell
kubectl expose deployment <deployment_name> --type=<NodePort|ClusterIP> --port=<container_port>

# Ví dụ
kubectl expose deployment guestbook --type=ClusterIP --port=3000
kubectl expose deployment guestbook --type=NodePort --port=30000
```

## Prompt trên chat GPT

Cú pháp
manifest cho việc expose service `tên services` cho `workload` `tên workload` với với dang `type_service`, container port `port_container`
Ví dụ
```
manifest cho việc expose service test-service cho deployment guestbook với với dang ClusterIP, container port 3000
```


## Kiểm tra hoạt động của services

Ví dụ expose deployment guestbook như lab deployments.

Kiểm tra với command như sau:
```shell
$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
guestbook    NodePort    10.96.13.184   <none>        3000:30624/TCP   6s
```

Do services ta sẽ truy cập theo thông số <IP_Node>:<Port_node_port>

Để kiểm tra IP của node, sử dụng command sau:
```shell
kubectl get node -o wide
kubectl get node -o wide
NAME                            STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                         KERNEL-VERSION      CONTAINER-RUNTIME
cluster-ingress-control-plane   Ready    control-plane   13h   v1.27.3   172.18.0.2    <none>        Debian GNU/Linux 11 (bullseye)   5.15.0-89-generic   containerd://1.7.1
cluster-ingress-worker          Ready    <none>          13h   v1.27.3   172.18.0.3    <none>        Debian GNU/Linux 11 (bullseye)   5.15.0-89-generic   containerd://1.7.1
```

Ta sẽ nhìn thấy IP của node lần lượt là 172.18.0.2 và 172.18.0.3

Đứng từ server ta thử truy cập `curl http://172.18.0.2:30624`

Truy cập từ external.

Do đang sử dụng Kind làm labs, và cấu hình kind đang là port 30000 nên ta cần edit nodeport về port 30000.
```shell
kubectl patch svc guestbook -p '{"spec":{"ports": [{"nodePort":30000, "port":3000,"protocol": "TCP", "targetPort":3000}]}}'
```

Kiểm tra lại
```shell
$ kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
guestbook    NodePort    10.96.13.184   <none>        3000:30000/TCP   5m19s
```

Ở client ta sẽ truy cập <IP_VM>:30000