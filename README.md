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


## Step 2: Ingress Controller Setup and Create MariaDB secrets and YAML Manifests
### 2.1 NGINX Ingress Installation
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
kubectl get pods -n ingress-nginx
```

### 2.2 Open ports in VPS Firewall
- Get the nodeport (30000-32767) for HTTP
```
kubectl get svc -n ingress-nginx
```
- Open the firewall ports
```
sudo ufw allow <NODEPORT>
sudo ufw allow 80
sudo ufw allow 443
```

### 2.3 Secret for MariaDB credentials:
```
kubectl create secret generic mariadb-secrets --namespace dev \
  --from-literal=db-user='YOUR-USER' \
  --from-literal=db-password='YOUR-PASSWORD' \
  --from-literal=db-host='mariadb-service.dev' \
  --from-literal=db-database='DATABASE-NAME' \
  --from-literal=app-url='http://dev.domain.com'

```

### 2.4 Manifest files for MariaDB (dev Namespace)

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
          value: db-database
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
```
kubectl get pods -n dev
kubectl get pvc -n dev
kubectl get svc -n dev
```

## Step 3: Develop and Containerize Laravel API

### 3.1 Install PHP and Composer
```
apt-get update
apt-get install -y php8.2 php8.2-cli php8.2-mbstring php8.2-mysql php8.2-bcmath php8.2-zip php8.2-curl php8.2-xml unzip
curl -sS https://getcomposer.org/installer | php
mv composer.phar /usr/local/bin/composer
```

### 3.2 Create the Laravel Backend

- Create the /backend directory and install Laravel
```
mkdir -p backend
cd backend
composer create-project laravel/laravel .
php artisan --version
```

- Copy .env.example to .env
```
cp .env.example .env
```

- Edit .env with the MariaDB credentials
```
sed -i 's/DB_CONNECTION=.*/DB_CONNECTION=mysql/' .env
sed -i 's/DB_HOST=.*/DB_HOST=db-host/' .env
sed -i 's/DB_PORT=.*/DB_PORT=3306/' .env
sed -i 's/DB_DATABASE=.*/DB_DATABASE=db-database/' .env
sed -i 's/DB_USERNAME=.*/DB_USERNAME=db-user/' .env
sed -i 's/DB_PASSWORD=.*/DB_PASSWORD=db-password/' .env
```

- Generate the app key:
```
php artisan key:generate
```

- Create Dockerfile to containerize Laravel
```
cat << EOF > Dockerfile
# Use PHP 8.1 with FPM for Laravel
FROM php:8.2-fpm

# Install system dependencies
RUN apt-get update && apt-get install -y \
    libpng-dev libjpeg-dev libfreetype6-dev libxml2-dev \
    libzip-dev unzip git \
    && docker-php-ext-install pdo_mysql gd xml zip

# Install Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Set working directory
WORKDIR /var/www

# Copy application code
COPY . .

# Install Laravel dependencies
RUN composer install --optimize-autoloader --no-dev

# Set permissions
RUN chown -R www-data:www-data /var/www/storage /var/www/bootstrap/cache

# Expose port for PHP-FPM
EXPOSE 9000

# Start PHP-FPM
CMD ["php-fpm"]
EOF
```

- Create a .dockerignore 
```
echo -e "vendor\nnode_modules\n.env" > .dockerignore
```

- Build and push the container
```
docker build -t your-username/laravel:dev .
docker push your-username/laravel:dev
```

### 3.3 Kubernetes Manifest files for Laravel (dev Namespace)

- Laravel Deployment (k8s/dev/laravel-deployment.yaml). Replace 'YOUR-USERNAME'

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel
  namespace: dev
spec:
  selector:
    matchLabels:
      app: laravel
  template:
    metadata:
      labels:
        app: laravel
    spec:
      containers:
      - name: laravel
        image: YOUR-USERNAME/laravel:dev
        env:
        - name: APP_URL
          valueFrom:
            secretKeyRef:
              name: mariadb-secrets
              key: app-url
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: mariadb-secrets
              key: db-host
        - name: DB_DATABASE
          valueFrom:
            secretKeyRef:
              name: mariadb-secrets
              key: db-database
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: mariadb-secrets
              key: db-user
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mariadb-secrets
              key: db-password
        ports:
        - containerPort: 9000
        resources:
          limits:
            memory: "500Mi"
            cpu: "500m"
          requests:
            memory: "200Mi"
            cpu: "200m"
```
- Laravel Service (k8s/dev/laravel-service.yaml)

```
apiVersion: v1
kind: Service
metadata:
  name: laravel-service
  namespace: dev
spec:
  selector:
    app: laravel
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9000
  type: ClusterIP
```

- Deploy to Kubernetes:
```
kubectl apply -f k8s/dev/laravel-config.yaml -n dev
kubectl apply -f k8s/dev/laravel-deployment.yaml -n dev
kubectl apply -f k8s/dev/laravel-service.yaml -n dev
```

- Steps to verify
```
kubectl get pods -n dev
kubectl get svc -n dev
```


### 3.4 Create the Vue Frontend

- Node.js and Vue CLI 
```
apt-get update
apt-get install -y nodejs npm
npm install -g @vue/cli
```

- Create the Vue.js app (Vue 3 - Babel, ESLint)
```
mkdir frontend
cd frontend
vue create . --no-git
```

- Docker container for Vue.js
```
cat << EOF > Dockerfile
FROM node:18 as build
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF
```

- Create .dockerignore 
```
echo -e "node_modules\nnpm-debug.log\ndist\n.env" > .dockerignore
```

- Build and push the container
```
docker build -t YOUR-USERNAME/frontend:dev .
docker push YOUR-USERNAME/frontend:dev
```

### 3.5 Kubernetes Manifest files for Frontend Vue.js App (dev Namespace)

- ConfigMap (k8s/dev/frontend-config.yaml)
```
cat << EOF > k8s/dev/frontend-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
name: frontend-config
namespace: dev
data:
VUE_APP_API_URL: "http://laravel-service.dev/laravel"
EOF
```

- Frontend deployment (k8s/dev/frontend-deployment.yaml)
```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: dev
spec:
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: buffyxo/frontend:dev
        env:
        - name: VUE_APP_API_URL
          valueFrom:
            configMapKeyRef:
              name: frontend-config
              key: VUE_APP_API_URL
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: "300Mi"
            cpu: "300m"
          requests:
            memory: "100Mi"
            cpu: "100m"

```

- Service (k8s/dev/frontend-service.yaml)
```
cat << EOF > k8s/dev/frontend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: dev
spec:
  selector:
    app: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
EOF
```

- Apply the manifests and deploy to Kubernetes
```
kubectl apply -f k8s/dev/frontend-config.yaml -n dev
kubectl apply -f k8s/dev/frontend-deployment.yaml -n dev
kubectl apply -f k8s/dev/frontend-service.yaml -n dev
```

- Verify that the pods and services are created and running
```
kubectl get pods -n dev
kubectl get svc -n dev
```