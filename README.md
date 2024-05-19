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


Step 2: Install K8S in Ubuntu 22.04, Take 2 instance 1-control plane, 1-worker node.

## Install commands in both control-plane & Node
```bash
vi script.sh && chmod +x script.sh

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

## Install commands in only control-plane
```bash
sudo kubeadm init

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl cluster-info
kubectl get nodes

kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml

sudo systemctl restart kubelet.service

kubectl get pods -n kube-system

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

          or
kubectl patch deployment metrics-server -n kube-system --type 'json' -p '[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'

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

