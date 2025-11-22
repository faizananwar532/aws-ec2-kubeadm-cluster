# AWS EC2 Kubeadm Cluster

## Kubernetes Cluster Setup on AWS with Kubeadm

This guide walks through setting up a Kubernetes cluster on AWS using EC2 instances and configuring it with Kubeadm.

---

## 1. AWS Infrastructure Setup

### **1.1. Create a VPC**
```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16
```
Note the `VpcId` from the response.

### **1.2. Create Subnets**
```bash
aws ec2 create-subnet --vpc-id <vpc-id> --cidr-block 10.0.1.0/24
```

### **1.3. Create an Internet Gateway**
```bash
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --internet-gateway-id <internet-gateway-id> --vpc-id <vpc-id>
```

### **1.4. Configure a Route Table**
```bash
aws ec2 create-route-table --vpc-id <vpc-id>
aws ec2 create-route --route-table-id <route-table-id> --destination-cidr-block 0.0.0.0/0 --gateway-id <internet-gateway-id>
```
Associate the route table with subnets:
```bash
aws ec2 associate-route-table --route-table-id <route-table-id> --subnet-id <subnet-id>
```

### **1.5. Create Security Groups**
```bash
aws ec2 create-security-group --group-name k8s-sg --description "Kubernetes SG" --vpc-id <vpc-id>
```
Allow inbound traffic for Kubernetes(we are allowing all-traffic however for security purpose you should always allow traffic for the ports you are expecting traffic for):
```bash
aws ec2 authorize-security-group-ingress \
  --group-id <security-gourp-id> \
  --protocol all \
  --port all \
  --cidr 0.0.0.0/0
```
Allow SSH(For certain protocol you can simply do it like this):
```bash
aws ec2 authorize-security-group-ingress --group-id <security-gourp-id> --protocol tcp --port 22 --cidr 0.0.0.0/0
```

### **1.6. Launch EC2 Instances**
You need key pair for connecting your ec2 instances. If the key pair doesn't exist, you can create it using: or using the GUI:
```bash
aws ec2 create-key-pair --key-name my-key --query 'KeyMaterial' --output text > my-key.pem
```

#### Get the aws ami lists
```bash
aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
            "Name=architecture,Values=x86_64" \
            "Name=virtualization-type,Values=hvm" \
            "Name=root-device-type,Values=ebs" \
  --query 'Images[*].[ImageId,Name]' \
  --output table \
  | sort -k2 -r | head -n 10
```
#### Master node
```bash
aws ec2 run-instances \
  --image-id ami-0fc5d935ebf8bc3bc \
  --count 1 \
  --instance-type t2.medium \
  --key-name <key-pair-name> \
  --subnet-id <subnet-id> \
  --security-group-ids <security-group-id> \
  --associate-public-ip-address

```
#### Worker node
```bash
aws ec2 run-instances \
  --image-id ami-0fc5d935ebf8bc3bc \
  --count 2 \
  --instance-type t2.small \
  --key-name <key-pair-name> \
  --subnet-id <subnet-id> \
  --security-group-ids <security-group-id> \
  --associate-public-ip-address
```
Retrieve instance details:
```bash
aws ec2 describe-instances --query "Reservations[*].Instances[*].[InstanceId,PrivateIpAddress]"
```

Connect with all master and worker nodes using your private key.
## 2. Kubeadm Installation on EC2 Instances
## `On Master and Worker Nodes (Both) from 2.1 - 2.6`

### **2.1. Update Packages**
```bash
sudo apt update && sudo apt upgrade -y
```

### **2.2. Install Kubectl**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"

echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

chmod +x kubectl
mkdir -p ~/.local/bin
mv ./kubectl ~/.local/bin/kubectl
```

### **2.3. Disable Swap**
```bash
sudo swapoff -a
```

### **2.4. Configure Parameters**
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

### **2.5. Install CRIO Runtime**
```bash
sudo apt-get update -y
sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates gpg

sudo curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" | sudo tee /etc/apt/sources.list.d/cri-o.list

sudo apt-get update -y
sudo apt-get install -y cri-o

sudo systemctl daemon-reload
sudo systemctl enable crio --now
sudo systemctl start crio.service
```

### **2.6. Install Kubernetes Components**
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update -y
sudo apt-get install -y kubelet="1.29.0-*" kubectl="1.29.0-*" kubeadm="1.29.0-*"
sudo apt-get update -y
sudo apt-get install -y jq

sudo systemctl enable --now kubelet
sudo systemctl start kubelet
```

## `Master Node only`
### **2.7. Initialize the Kubernetes Master Node**
```bash
sudo kubeadm config images pull

sudo kubeadm init

mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config

# Network Plugin = calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
```
Generate a token for worker nodes:
```bash
kubeadm token create --print-join-command
```

## `Worker Node only`
### **2.8. Join Worker Nodes**
```bash
sudo kubeadm reset pre-flight checks
```
copy the output of this command from master node: `kubeadm token create --print-join-command` with sudo and `--v=5` append at the end, and paste it in worker node to attach it to master plan or run the command in following order:
```bash
sudo kubeadm join <MasterPrivateIP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH> --v=5
```

## 3. Deploying Applications with Affinity and Anti-Affinity

### **Deploy Service A (Anti-Affinity)**
Create `service-a.yaml`: in your master node do the following:
> `nano service-a.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-a
spec:
  replicas: 2
  selector:
    matchLabels:
      app: service-a
  template:
    metadata:
      labels:
        app: service-a
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - service-a
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: http-echo
        image: hashicorp/http-echo:latest
        args: ["-text", "Hello from service-a"]

---
apiVersion: v1
kind: Service
metadata:
  name: service-a-lb
spec:
  type: ClusterIP  # Change this from LoadBalancer to ClusterIP
  ports:
  - port: 80
    targetPort: 5678
  selector:
    app: service-a
```
Apply deployment:
```bash
kubectl apply -f service-a.yaml
```

### **Deploy Service B (Affinity)**
Create `service-b.yaml`:
> `nano service-b.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: service-b
spec:
  replicas: 2
  selector:
    matchLabels:
      app: service-b
  template:
    metadata:
      labels:
        app: service-b
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - service-b
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: http-echo
        image: hashicorp/http-echo:latest
        args: ["-text", "Hello from service-b"]

---
apiVersion: v1
kind: Service
metadata:
  name: service-b-lb
spec:
  type: ClusterIP  # Change this from LoadBalancer to ClusterIP
  ports:
  - port: 80
    targetPort: 5678
  selector:
    app: service-b
```
Apply deployment:
```bash
kubectl apply -f service-b.yaml
```

### **Check Pods Distribution**
```bash
kubectl get pods -o wide
```

### **Test Services**
```bash
kubectl run curl-test --image=curlimages/curl --rm -it -- sh
curl http://service-a-lb:80
curl http://service-b-lb:80
```

### **Cleanup**
```bash
kubectl delete -f service-a.yaml
kubectl delete -f service-b.yaml
```

Your Kubernetes cluster is now fully set up with AWS, Kubeadm, and a deployment using node affinity and anti-affinity rules! 🚀