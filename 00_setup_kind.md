# KIND

Kind (Kubernetes in Docker) sử dụng làm môi trường labs cho các services chạy trên Docker

Để setup Kind, đầu tiên trên Host(Virtual Machine hoặc Physical Machine) phải cài docker

## Setup docker

Ubuntu
``` shell
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
apt update
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## Setup kind

```shell
wget https://github.com/kubernetes-sigs/kind/releases/download/v0.20.0/kind-linux-amd64
chmod +x kind-linux-amd64
mv kind-linux-amd64 /usr/local/bin/kind
```

## Setup kubectl

```shell
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

## Initilize cluster kubernetes with Kind

```shell
cat << EOF > cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cluster-kind
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
- role: worker
- role: worker
EOF

kind create cluster --config=cluster.yaml
```

Đối với khai báo yaml ở trên, kind sẽ create ra 4 containers tương tự như 4 host ảo với trong 1 cluster Kubernetes. Trong config trên, sẽ có 1 master và 3 worker

Chạy các command sau để check thông tin cluster

```shell

# Check docker
docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS          PORTS                                                                 NAMES
b4d9f7189eeb   kindest/node:v1.27.3   "/usr/local/bin/entr…"   25 minutes ago   Up 25 minutes                                                                         cluster-ingress-worker3
ac9569a8254a   kindest/node:v1.27.3   "/usr/local/bin/entr…"   25 minutes ago   Up 25 minutes   0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 127.0.0.1:41611->6443/tcp   cluster-ingress-control-plane
f93b37a8d1c6   kindest/node:v1.27.3   "/usr/local/bin/entr…"   25 minutes ago   Up 25 minutes                                                                         cluster-ingress-worker2
51383969b565   kindest/node:v1.27.3   "/usr/local/bin/entr…"   25 minutes ago   Up 25 minutes                                                                         cluster-ingress-worker

# Check cluster
kubectl get node
NAME                            STATUS   ROLES           AGE   VERSION
cluster-ingress-control-plane   Ready    control-plane   25m   v1.27.3
cluster-ingress-worker          Ready    <none>          25m   v1.27.3
cluster-ingress-worker2         Ready    <none>          25m   v1.27.3
cluster-ingress-worker3         Ready    <none>          25m   v1.27.3
```

## Install helm

```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

## Delete cluster

```shell
kind delete cluster --name <name_cluster>
# Eg:
kind delete cluster --name cluster-ingress
```