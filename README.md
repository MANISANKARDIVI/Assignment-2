# Assignment-2

NodeJS application horizontalAutoScaling (hpa) by using K8S

Step 1: Download the assignment files from this link https://drive.google.com/file/d/12gZcH1Jr1xWWRzyH3yMqJ9DV_3rlHsnK/view?usp=sharing
## The assignment has two files-
  - main.js - Simple Node/Express REST API that returns a “Hello, World!” response on port 3000
  - package.json - Dependency management file
##

## Previously i have done Docker file.
  - GitHub URL:-  git clone https://github.com/MANISANKARDIVI/Assignment.git
##


Step 2: Install K8S in Ubuntu 20.04, Take 2 instance 1-control-plane, 1-node.

## Install commands in both control-plane & Node
```bash
vi script.sh && chmod +x script.sh

sudo -i
sudo apt update -y && sudo apt upgrade -y
sudo apt install docker.io -y
sudo chmod 666 /var/run/docker.sock
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubeadm kubelet kubectl kubernetes-cni
```

## Install commands in only control-plane
```bash
sudo kubeadm init

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

Note: This is a sample token ==> kubeadm join 192.168.56.110:6443 --token hp9b0k.1g9tqz8vkf78ucwf     --discovery-token-ca-cert-hash sha256:32eb67948d72ba99aac9b5bb0305d66a48f43b0798cb2df99c8b1c30708bdc2c

copy paste generated token in your worker node.

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml

sudo systemctl restart kubelet.service

kubectl get nodes
```
## Docker Build the Image
```bash
git clone https://github.com/MANISANKARDIVI/Assignment.git

cd Assignment

docker build -t dockerUsername/imagename .

docker images

docker login

username = Enter your Docker username
password = Enter your Docker password

docker push dockerUsername/imagename

check once in your DockerHub
```
## Run comands in Control-plane
```bash
kubectl get pods
kubectl get node
kubectl get svc
```

## Install Metric-server in contaol-plane

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl get deployment metrics-server -n kube-system

If metric-server is created then fine otherwise follow below commands:-

kubectl get apiservices.apiregistration.k8s.io

kubectl get all -n kube-system

kubectl get po -n kube-system

kubectl edit deploy metrics-server -n kube-system

add this line: in 44 line below

- --kubelet-insecure-tls=true

now check the metric-server is installed or not.

kubectl get po -n kube-system
kubectl get apiservices.apiregistration.k8s.io
kubectl get deployment metrics-server -n kube-system

```

## now Create deployment.yaml file

```bash
vi deployment.yaml

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hpa-deployment
  template:
    metadata:
      labels:
        app: hpa-deployment
    spec:
      containers:
      - name: node-app-cont
        image: Enter-your-imagname  <=======
        ports:
        - name: http
          containerPort: 3000
        resources:
          requests:
            memory: "64Mi" # Request 64 MiB of memory
            cpu: "100m" # Request 250 millicpu (0.25 CPU)
          limits:
            memory: "128Mi" # Limit memory usage to 128 MiB
            cpu: "500m" # Limit CPU usage to 500 millicpu (0.5 CPU)

kubectl apply -f deployment.yaml

kubectl get deploy

```
## now Create service.yaml file

```bash
vi service.yaml

---
apiVersion: v1
kind: Service
metadata:
  name: hpa-deployment-svc
spec:
  type: NodePort
  selector:
    app: hpa-deployment
  ports:
  - port: 3000 # Expose port 3000
    targetPort: 3000 # Target port 3000 on pods

kubectl apply -f service.yaml

kubectl get svc -o wide

```

## now Create hpa.yaml file

```bash
vi hpa.yaml

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-deployment
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-deployment
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50

kubectl apply -f hpa.yaml

kubectl get hpa

```
## now Run the commands contol-plane
```bash
kubectl get pods -o wide
kubectl get nodes -o wide
kubectl get svc -o wide

kubectl get hpa

kubectl top nodes
kubectl top pods

we can check copy publicIP of node-1 and paste in  browser and check svc nodePortIP
example ==> http://18.61.33.190:30413

```
## now Test the load in pods

```bash
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://publicIP:NodePortIP; done"

Take new terminal and check pods

kubectl get pods

kubectl get hpa

kubectl describe hpa hpa-pod-name 
```
We can see Replicas as Generated 
after press ctrl+c for end the load generator 

we can check it will take 3 to 5 mins to scale down the generated pods,

```bash
kubectl get pods

kubectl get hpa

kubectl describe hpa hpa-pod-name 
```

## Demo

Video Link for Assignment-2
https://drive.google.com/file/d/13mMnpD80zUtAEbx35jb7kQXHlD0nnm1D/view?usp=sharing

screenshots for assignment-2
https://drive.google.com/drive/folders/11ywzL1Mb0LrQq5I9jxyly_I97GoeTKpw?usp=drive_link

