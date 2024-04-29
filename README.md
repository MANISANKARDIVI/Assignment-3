# Assignment-3

Add PostgreSQL to the cluster. Also add Prometheus, and Grafana to monitor PostgreSQL and ensure data persistence for all of them that will prevent data loss during crashes/restarts.

Step 1: 

## Previous Assignment-2 file.
  - GitHub URL:-  git clone https://github.com/MANISANKARDIVI/Assignment-2.git
##

Step 2: Install K8S in Ubuntu 22.04, Take 2 instance T2.small, 1-control-plane, 1-workernode.

## Install commands in both control-plane & worker-Node
```bash
sudo -i
sudo apt update -y && sudo apt upgrade -y
sudo reboot

sudo apt install docker.io -y
sudo usermod -aG docker $USER
sudo chmod 777 /var/run/docker.sock
systemctl enable docker
```
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/kubernetes.conf <<EOT
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOT

sudo sysctl --system

sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg

sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt update
sudo apt install -y containerd.io

containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1

sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
## Install commands in control-plane
```bash
sudo kubeadm init

sudo mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl cluster-info

kubectl get nodes

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

kubectl get pods -n kube-system
```

## Deployment of postgresql
```bash
$ vi pgdeploy.yaml

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  selector:
   matchLabels:
    app: postgres
  replicas: 1
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:14
          imagePullPolicy: "IfNotPresent"
          envFrom:
            - configMapRef:
                name: postgres-config
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgresdb
      volumes:
        - name: postgresdb
          persistentVolumeClaim:
            claimName: postgres-pv-claim


$ kubectl apply -f pgdeploy.yaml

```
## service of postgresql
```bash
$ vi service.yaml

---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  ports:
    - name: postgres
      port: 5432
      nodePort: 30432
  type: NodePort
  selector:
    app: postgres


$ kubectl apply -f service.yaml 
```

## config of postgresql
```bash

$ vi config.yaml

---
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app: postgres
  name: postgres-config
data:
  POSTGRES_DB: postgresdb
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: root


$ kubectl apply -f config.yaml

```

## pv of postgresql
```bash

$ vi persistentVolume.yaml

---
apiVersion: v1
kind: PersistentVolume
metadata:
  labels:
    app: postgres
    type: local
  name: postgres-pv-volume
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 1Gi
  hostPath:
    path: /mnt/data
  storageClassName: manual

$ kubectl apply -f persistentVolume.yaml

```
## pvc of postgresql
```bash
$ vi persistentVolumeClaim.yaml

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels:
    app: postgres
  name: postgres-pv-claim
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: manual
  
$ kubectl apply -f persistentVolumeClaim.yaml
```

```bash
kubectl get pv

kubectl get pvc

so we can create a file in Postgres pod:- 

$ kubectl exec -it postgres-b84fd8466-89wwq -- /bin/bash

so we are inside of the container

$ ls 

$ cd /var/lib/postgresql/data

$ mkdir mani 

$ cd mani

$ touch file.txt

$ exit

$ kubectl delete pod pod-name

so new pod will be created with diff name

$ kubectl get pods

$ kubectl exec -it enter-pod-name -- /bin/bash

$ cd /var/lib/postgresql/data

we can check if pod is deleted but data is present

we can conect to db and create table :-

psql -d postgresdb -U postgres


password = root

List available databases

$ \l

List available tables

$ \dt

CREATE TABLE cars (
  brand VARCHAR(255),
  model VARCHAR(255),
  year INT
);


INSERT INTO cars (brand, model, year)
VALUES
  ('Volvo', 'p1800', 1968),
  ('BMW', 'M1', 1978),
  ('Toyota', 'Celica', 1975);

SELECT * FROM cars;

To exit form db

$ \q

by this persistentVolumeClaim is successfully implemented.
```

## now we need to implement grafana & prometheus & helm

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

```bash
helm repo add stable https://charts.helm.sh/stable

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo add grafana https://grafana.github.io/helm-charts

helm repo update

kubectl create namespace prometheus

helm install stable prometheus-community/kube-prometheus-stack -n prometheus

helm repo update

helm search repo prometheus-community

kubectl edit svc stable-grafana -n prometheus 

kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
```


## Grafana
username: admin
passwd: prom-operator

```bash
Cluster Monitoring = 15757
Namespace monitoring = 15758
Node Monitoring = 15759 , 1860
Pod level monitoring = 5225
PVC usage - 13646
```

## Demo

Video Link for Assignment-3
https://drive.google.com/drive/folders/1Q4wfWvqxcmt-id1UaHHLV208goBozJCq?usp=sharing


