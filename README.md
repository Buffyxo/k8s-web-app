# Kubernetes Web App
This project demonstrates the deployment of a full-stack web application on a single-node Kubernetes cluster using `kubeadm`, running on a 3.5GB Debian 12 VPS.

These are the web application components which will be containerized and deployed as separate pods:
- Frontend → Vue
- Backend → REST API built with Laravel
- Database → MariaDB

In this project, I will showcase my progress as a brief step-by-step documentation in an effort to learn Kubernetes for a production-style environment. The infrastructure will be built at pod-level with future plans to scale it to a multi-node cluster for further experimentation. The full instructions will be added to my blog once everything is working. 

## Step 1: VPS and Kubernetes Cluster Setup

### 1.1 Initial VPS settings
- Install necessary packages and disable swap
```
apt-get update && apt-get upgrade
apt-get install curl apt-transport-https ca-certificates gnupg
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
```

- Kernel modules and settings
```
modprobe overlay
modprobe br_netfilter
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.bridge.bridge-nf-call-iptables=1
echo -e "net.ipv4.ip_forward=1\nnet.bridge.bridge-nf-call-iptables=1" | tee /etc/sysctl.d/k8s.conf
sysctl --system
```


### 1.2 Setting up Docker
- Install `docker-ce`, `docker-ce-cli`, `containerd.io`.
```
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian bookworm stable" | tee /etc/apt/sources.list.d/docker.list
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io=1.6.21-1
```
- **Required for Kubernetes:** Configure `systemd` cgroup driver in `/etc/docker/daemon.json`.
```
mkdir -p /etc/docker
echo '{"exec-opts": ["native.cgroupdriver=systemd"]}' | tee /etc/docker/daemon.json
systemctl restart docker
systemctl enable docker
```
- Add user to Docker group.
```
usermod -aG docker $USER
```


### 1.3 Kubernetes Installation
- Add Kubernetes repository and install `kubeadm`, `kubelet`, `kubectl`.
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install kubeadm kubelet kubectl
```

### 1.4 Cluster Initialization and Setup
- Initialize:
```
kubeadm init --pod-network-cidr=10.244.0.0/16
```

- Set up `kubectl`:
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
- Allow pods on master:
```
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```
#### Possible Errors:
---
###### **If CRI error occur, follow these steps:**
######  Verify CRI endpoint
```
crictl -r unix:///var/run/containerd/containerd.sock info

```
###### 1. Generate config.toml if missing:
```
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
```
###### 2. Ensure that CRI is not disabled (remove cri from `disabled_plugins = ["cri"]`).
```
nano /etc/containerd/config.toml
```
###### 3. Find `[plugins."io.containerd.grpc.v1.cri"]` and ensure these before verifying CRI endpoint again (should return JSON output if successful):
```
[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "registry.k8s.io/pause:3.9"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
      SystemdCgroup = true

```
---
###### **Other possible fixes:**
###### - If errors persist, try resetting kubeadm and follow the Cluster Initialization and Setup steps again
```
kubeadm reset -f
```
###### - Check memory usage and that there is enough free memory (ensure memory is not cached):
###### *APIs and containers may stop working when memory is unavailable*
```
free -m
sync; echo 3 | tee /proc/sys/vm/drop_caches
```
###### - Additional memory management:
```
nano /etc/sysctl.conf
```
###### Add:
```
vm.vfs_cache_pressure=200
vm.swappiness=10
vm.dirty_ratio=10
vm.dirty_background_ratio=5
```
---

### 1.5 Install Container Network Interface (CNI) Plugin - Using Flannel
```
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/v0.22.0/Documentation/kube-flannel.yml
kubectl get pods -n kube-flannel
```

### 1.6 Create and verify namespaces for prod and dev
```
kubectl create namespace dev
kubectl create namespace prod
kubectl get namespaces
```

### 1.7 Check cluster health
```
kubectl get nodes
kubectl get pods -A
```

### 1.8 Security and Sensitive Files
- Included in .gitignore: `admin.conf`, `.kube/config`


## Step 2: Ingress Controller Setup and Create YAML Manifests
### 2.1 NGINX Ingress Installation
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
kubectl get pods -n ingress-nginx
```

### 2.2 Open ports in VPS Firewall
- Get the nodeport (30000-32767)
```
kubectl get svc -n ingress-nginx
```
- Open the nodeport
```
sudo ufw allow <NODEPORT>
sudo ufw allow 80
sudo ufw allow 443
```

### 2.3 Secret for MariaDB credentials:
```
kubectl create secret generic mariadb-secrets --namespace dev \
  --from-literal=db-user='YOUR-USER' \
  --from-literal=db-password='YOUR-PASSWORD'
```

### 2.4 Manifest files
- MariaDB PVC (k8s/dev/mariadb-pvc.yaml)
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb-pvc
  namespace: dev
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-path
  ```

  - MariaDB Deployment (k8s/dev/mariadb-deployment.yaml)
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb
  namespace: dev
spec:
  selector:
    matchLabels:
      app: mariadb
  template:
    metadata:
      labels:
        app: mariadb
    spec:
      containers:
      - name: mariadb
        image: mariadb:10.11
        env:
        - name: MYSQL_USER
          valueFrom:
            secretKeyRef:
              name: mariadb-secrets
              key: db-user
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secrets
              key: db-password
        - name: MYSQL_DATABASE
          value: blog_dev
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secrets
              key: db-password
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mariadb-data
          mountPath: /var/lib/mysql
        resources:
          limits:
            memory: "500Mi"
            cpu: "500m"
          requests:
            memory: "200Mi"
            cpu: "200m"
      volumes:
      - name: mariadb-data
        persistentVolumeClaim:
          claimName: mariadb-pvc
```

- MariaDB Service (k8s/dev/mariadb-service.yaml)
```
apiVersion: v1
kind: Service
metadata:
  name: mariadb-service
  namespace: dev
spec:
  selector:
    app: mariadb
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306
  type: ClusterIP
```

- Create local storage class for the PVC:
```
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

- Apply Manifests
```
kubectl apply -f k8s/dev/mariadb-pvc.yaml -n dev
kubectl apply -f k8s/dev/mariadb-deployment.yaml -n dev
kubectl apply -f k8s/dev/mariadb-service.yaml -n dev
```

- Verify that the MariaDB pod is `running`, the PVC shows `bound`, and that the MariaDB service is accessible within the Kubernetes cluster (10.96.0.0/12 range).