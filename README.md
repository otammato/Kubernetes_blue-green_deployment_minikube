# Kubernetes_blue-green_deployment_minikube
[ Kubernetes ] Deploy a simple app using a blue-green deployment strategy

<img width="330" alt="Screenshot 2023-03-01 at 14 38 28" src="https://user-images.githubusercontent.com/104728608/222228704-d2daddb5-a8f6-4c96-9cd9-b61421466ba8.png"> <img width="330" alt="Screenshot 2023-03-01 at 14 38 28" src="https://user-images.githubusercontent.com/104728608/222230138-9bf46875-2a2a-41bd-936c-d35df1f35935.png"><&nbsp><img width="330" alt="Screenshot 2023-03-01 at 14 38 28" src="https://user-images.githubusercontent.com/104728608/222230833-e6c06414-1bd7-4b12-9ceb-1fae96495b89.png">


### 1. Installing and starting Kubernetes, Docker, kubectl and minicube

<details markdown=1><summary markdown="span">Set of commands</summary>

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
</details>

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

service-live.yaml

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: k8s-boot-demo-service
spec:
  type: NodePort
  selector:
    app: k8s-boot-demo
    version: v1
  ports:
    - name: app-port-mapping
      protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 30090
```


```
kubectl apply -f deployment-blue.yaml
kubectl apply -f service-live.yaml
minikube ip
192.168.49.2
curl 192.168.49.2:30090/api/info
```

<img width="1024" alt="Screenshot 2023-03-01 at 15 32 03" src="https://user-images.githubusercontent.com/104728608/222186758-f5a1ccbc-5f98-461a-951c-d2c8e63cbabb.png">


deployment-green.yaml

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-boot-demo-deployment-green
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: k8s-boot-demo
      version: v2
      color: green
  template:
    metadata:
      labels:
        app: k8s-boot-demo
        version: v2
        color: green
    spec:
      containers:
        - name: k8s-boot-demo
          image: sivaprasadreddy/k8s-boot-demo:v2
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
```

service-preprod.yaml

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: k8s-boot-demo-service-preprod
spec:
  type: NodePort
  selector:
    app: k8s-boot-demo
    version: v2
  ports:
    - name: app-port-mapping
      protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 30092
```

```
kubectl apply -f deployment-green.yaml
kubectl apply -f service-preprod.yaml
kubectl get all
minikube ip
192.168.49.2
curl 192.168.49.2:30090/api/info
{"hostName":"k8s-boot-demo-deployment-blue-7459fc4bd8-rk8kt","app":"K8S SpringBoot Demo","version":"v1"}
$ curl 192.168.99.103:30092/api/info
{"version":"v2","app":"K8S SpringBoot Demo","hostName":"k8s-boot-demo-deployment-green-d7b94fdc5-5xxgw"}
```

<img width="1024" alt="Screenshot 2023-03-01 at 15 49 31" src="https://user-images.githubusercontent.com/104728608/222191512-33271011-a36e-4d44-8772-8b52407f2fd0.png">

service-live.yaml

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: k8s-boot-demo-service
spec:
  type: NodePort
  selector:
    app: k8s-boot-demo
    version: v2
  ports:
    - name: app-port-mapping
      protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 30090
```

<img width="1024" alt="Screenshot 2023-03-01 at 16 16 00" src="https://user-images.githubusercontent.com/104728608/222199260-2aff1fea-1bb2-4272-bdd5-c36753ba1c18.png">

``` 
kubectl apply -f service-live.yaml
minikube ip
192.168.49.2
curl 192.168.49.2:30090/api/info
curl 192.168.49.2:30092/api/info
```

<img width="1024" alt="Screenshot 2023-03-01 at 16 04 42" src="https://user-images.githubusercontent.com/104728608/222195794-15e3da98-2862-46a7-8f77-e9b3dc8ee11b.png">


