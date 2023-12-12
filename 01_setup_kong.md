# Setup Kong

```shell
$ kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml

$ helm repo add kong https://charts.konghq.com
$ helm repo update
$ kubectl create ns kong
$ helm -n kong install kong kong/ingress


```


```json
{
  "spec": {
    "replicas": 1,
    "template": {
      "spec": {
        "containers": [
          {
            "name": "proxy",
            "ports": [
              {
                "containerPort": 8000,
                "hostPort": 80,
                "name": "proxy",
                "protocol": "TCP"
              },
              {
                "containerPort": 8443,
                "hostPort": 443,
                "name": "proxy-tls",
                "protocol": "TCP"
              }
            ]
          }
        ],
        "nodeSelector": {
          "ingress-ready": "true"
        },
        "tolerations": [
          {
            "key": "node-role.kubernetes.io/control-plane",
            "effect": "NoSchedule"
          }
        ]
      }
    }
  }
}
```

```shell
kubectl patch deployment -n kong kong-gateway -p '{"spec":{"replicas":1,"template":{"spec":{"containers":[{"name":"proxy","ports":[{"containerPort":8000,"hostPort":80,"name":"proxy","protocol":"TCP"},{"containerPort":8443,"hostPort":443,"name":"proxy-tls","protocol":"TCP"}]}],"nodeSelector":{"ingress-ready":"true"},"tolerations":[{"key":"node-role.kubernetes.io/control-plane","effect":"NoSchedule"}]}}}}'
```