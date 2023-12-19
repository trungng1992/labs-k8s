# Joget with Kubernetes (Linode)

## Create Nginx Class.

Mặc định kubernetes trên Linode chưa có Ingress.

### Tạo namespace cho ingress-nginx

```shell
$ kubectl create namespace ingress-nginx
```

### Install ingress-nginx bằng Helm

```shell
$ helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
$ helm repo update
$ cat << EOF > values.yaml
controller:
  config:
    enable-underscores-in-headers: "true" # Need for Joget
EOF
$ helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx -f values.yaml
```

## Install MySQL

```shell
$ cat << EOF > values-helm-db.yaml
global:
  storageClass: linode-block-storage
EOF
$ helm install mysql oci://registry-1.docker.io/bitnamicharts/mysql -f values-helm-db.yaml
```

```
# Result
NAME: mysql
LAST DEPLOYED: Mon Dec 18 23:20:17 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: mysql
CHART VERSION: 9.15.0
APP VERSION: 8.0.35

** Please be patient while the chart is being deployed **

Tip:

  Watch the deployment status using the command: kubectl get pods -w --namespace default

Services:

  echo Primary: mysql.default.svc.cluster.local:3306

Execute the following to get the administrator credentials:

  echo Username: root
  MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default mysql -o jsonpath="{.data.mysql-root-password}" | base64 -d)

To connect to your database:

  1. Run a pod that you can use as a client:

      kubectl run mysql-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:8.0.35-debian-11-r0 --namespace default --env MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD --command -- bash

  2. To connect to primary service (read/write):

      mysql -h mysql.default.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"
```

## Install LongHorn for PVC RWM
```shell
$ helm repo add longhorn https://charts.longhorn.io
$ helm repo update

$ kubectl create namespace longhorn-system
$ helm install longhorn longhorn/longhorn --namespace longhorn-system
```

## Install Joget

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: joget-dx7-tomcat9-pvc
spec:
  accessModes:
    - ReadWriteMany
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
  storageClassName: longhorn
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: joget-dx7-tomcat9
  labels:
    app: joget-dx7-tomcat9
spec:
  replicas: 1
  selector:
    matchLabels:
      app: joget-dx7-tomcat9
  template:
    metadata:
      labels:
        app: joget-dx7-tomcat9
    spec:
      volumes:
        - name: joget-dx7-tomcat9-pv
          persistentVolumeClaim:
            claimName: joget-dx7-tomcat9-pvc
      securityContext:
        runAsUser: 1000
        fsGroup: 0
      containers:
        - name: joget-dx7-tomcat9
          image: jogetworkflow/joget-dx7-tomcat9:latest
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: joget-dx7-tomcat9-pv
              mountPath: /opt/joget/wflow
          env:
            - name: KUBERNETES_NAMESPACE
              valueFrom:
                fieldRef:
                    fieldPath: metadata.namespace
---
apiVersion: v1
kind: Service
metadata:
  name: joget-dx7-tomcat9
  labels:
    app: joget-dx7-tomcat9
spec:
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: https
    port: 9080
    targetPort: 9080
  selector:
    app: joget-dx7-tomcat9
  type: NodePort
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: joget-dx7-tomcat9-clusterrolebinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view
subjects:
  - kind: ServiceAccount
    name: default
    namespace: default
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: joget-dx7-tomcat9-ingress
  annotations:
    nginx.ingress.kubernetes.io/affinity: cookie
    # nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: joget.idb.local
      http:
        paths:
          - path: /jw
            pathType: Prefix
            backend:
              service:
                name: joget-dx7-tomcat9
                port:
                  number: 8080
```

## Scale replica=2

kubectl scale deployment joget-dx7-tomcat9 --replicas=2

kubectl get pod
```
kubectl get pod
NAME                                 READY   STATUS    RESTARTS   AGE
joget-dx7-tomcat9-79cd85b47c-2x5bs   1/1     Running   0          47s
joget-dx7-tomcat9-79cd85b47c-zkzgf   1/1     Running   0          109s
mysql-0                              1/1     Running   0          10m
```