# Kubernetes_blue-green_deployment_minikube
[ Kubernetes ] Deploy a simple app using a blue-green deployment strategy


### 1. Installing and starting Kubernetes, Docker, kubectl and minicube

```
sudo su

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```
```
yum install -y kubectl
```
```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```
```
su ec2-user # switch to your standard user

sudo yum install docker

sudo usermod -aG docker $USER && newgrp docker

sudo systemctl enable docker

sudo systemctl start docker

sudo reboot
```
```
minikube start
```

<img width="1024" alt="Screenshot 2023-03-01 at 14 38 28" src="https://user-images.githubusercontent.com/104728608/222172364-fde0d4c1-5538-4b04-a480-03e3293b1e42.png">

### 2. 

deployment-blue.yaml

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-boot-demo-deployment-blue
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: k8s-boot-demo
      version: v1
      color: blue
  template:
    metadata:
      labels:
        app: k8s-boot-demo
        version: v1
        color: blue
    spec:
      containers:
        - name: k8s-boot-demo
          image: sivaprasadreddy/k8s-boot-demo:v1
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
```
