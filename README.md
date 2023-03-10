# Kubernetes_blue-green_deployment_minikube_demo
[ Kubernetes ] Deploy a simple app using a blue-green deployment strategy

> In Kubernetes there are a few different ways to release an application, you have
to carefully choose the right strategy to make your infrastructure resilient.

- [recreate](recreate/): terminate the old version and release the new one
- [ramped](ramped/): release a new version on a rolling update fashion, one
  after the other
- [blue/green](blue-green/): release a new version alongside the old version
  then switch traffic
- [canary](canary/): release a new version to a subset of users, then proceed
  to a full rollout
- [a/b testing](ab-testing/): release a new version to a subset of users in a
  precise way (HTTP headers, cookie, weight, etc.). This doesn’t come out of the
  box with Kubernetes, it imply extra work to setup a smarter
  loadbalancing system (Istio, Linkerd, Traeffik, custom nginx/haproxy, etc).
- [shadow](shadow/): release a new version alongside the old version. Incoming
  traffic is mirrored to the new version and doesn't impact the
  response.

<p align="center">
  <img width="700" alt="Screenshot 2023-03-01 at 14 38 28" src="https://user-images.githubusercontent.com/104728608/222240448-e97d7187-d243-402f-9bbe-c9448c835656.png">
</p>

<br>

## In this demo we will use the "blue-green" approach

<br>

<p align="center">
  <img width="250" alt="Screenshot 2023-03-01 at 14 38 28" src="https://user-images.githubusercontent.com/104728608/222228704-d2daddb5-a8f6-4c96-9cd9-b61421466ba8.png">  <img width="250" alt="Screenshot 2023-03-01 at 14 38 28" src="https://user-images.githubusercontent.com/104728608/222230138-9bf46875-2a2a-41bd-936c-d35df1f35935.png">  <img width="250" alt="Screenshot 2023-03-01 at 14 38 28" src="https://user-images.githubusercontent.com/104728608/222230833-e6c06414-1bd7-4b12-9ceb-1fae96495b89.png">
</p>

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

### 2. Launching Kubernetes deployments and services

#### 2.1. Launching the "blue" set

deployment-blue.yaml

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-boot-demo-deployment-blue
spec:
  replicas: 3
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

#### 2.2. Launching the "green" set

deployment-green.yaml

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-boot-demo-deployment-green
spec:
  replicas: 3
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
```

#### 2.3. Both deployments are active now:
```
curl 192.168.49.2:30090/api/info
{"hostName":"k8s-boot-demo-deployment-blue-7459fc4bd8-rk8kt","app":"K8S SpringBoot Demo","version":"v1"}
$ curl 192.168.99.103:30092/api/info
{"version":"v2","app":"K8S SpringBoot Demo","hostName":"k8s-boot-demo-deployment-green-d7b94fdc5-5xxgw"}
```

<img width="1024" alt="Screenshot 2023-03-01 at 15 49 31" src="https://user-images.githubusercontent.com/104728608/222191512-33271011-a36e-4d44-8772-8b52407f2fd0.png">

### 2.5. Patching the "blue" service to change the selector to v2 (which is used to specify the version of deployment) .

```
kubectl patch service service-live.yaml -p '{"spec":{"selector":{"version":"v2"}}}'
```

Alternatively, this "sed" command can be used to perform a global search and replace operation on the "service-live.yaml" file. The pattern to search for is "version: v1", and the replacement value is "version: v2". The "-i" option modifies the file in place.

```
# Find and replace the version value in the Service manifest
sed -i 's/version: v1/version: v2/g' service-live.yaml
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


### 3. When you make sure the green deployment works as expected, remove the "blue" deployment and the "pre-prod" service

```
kubectl delete -f service-preprod.yaml
```

```
kubectl delete -f deployment-blue.yaml
```
<img width="1024" alt="Screenshot 2023-03-04 at 15 11 12" src="https://user-images.githubusercontent.com/104728608/222913951-79aa18b7-ca87-45ac-84f6-f728c15a95c1.png">

