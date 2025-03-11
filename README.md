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
aws ec2 create-subnet --vpc-id <VpcId> --cidr-block 10.0.1.0/24
```

### **1.3. Create an Internet Gateway**
```bash
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway --internet-gateway-id <GatewayId> --vpc-id <VpcId>
```

### **1.4. Configure a Route Table**
```bash
aws ec2 create-route-table --vpc-id <VpcId>
aws ec2 create-route --route-table-id <RouteTableId> --destination-cidr-block 0.0.0.0/0 --gateway-id <GatewayId>
```
Associate the route table with subnets:
```bash
aws ec2 associate-route-table --route-table-id <RouteTableId> --subnet-id <SubnetId>
```

### **1.5. Create Security Groups**
```bash
aws ec2 create-security-group --group-name k8s-sg --description "Kubernetes SG" --vpc-id <VpcId>
```
Allow inbound traffic for Kubernetes(we are allowing all-traffic however for security purpose you should always allow traffic for the ports you are expecting traffic for):
```bash
aws ec2 authorize-security-group-ingress --group-id <GroupId> --protocol all --port all --source-group <GroupId>
```
Allow SSH(For certain protocol you can simply do it like this):
```bash
aws ec2 authorize-security-group-ingress --group-id <GroupId> --protocol tcp --port 22 --cidr 0.0.0.0/0
```

### **1.6. Launch EC2 Instances**
You need key pair for connecting your ec2 instances. If the key pair doesn't exist, you can create it using:
```bash
aws ec2 create-key-pair --key-name my-key --query 'KeyMaterial' --output text > my-key.pem
```
#### Master node
```bash
aws ec2 run-instances --image-id ami-0e56583ebfdfc098f --count 1 --instance-type t3.medium --key-name my-key --subnet-id <SubnetId> --security-group-ids <GroupId> --key-name my-key 
```
#### Worker node
```bash
aws ec2 run-instances --image-id ami-0e56583ebfdfc098f --count 2 --instance-type t3.small --key-name my-key --subnet-id <SubnetId> --security-group-ids <GroupId> --key-name my-key 
```
Retrieve instance details:
```bash
aws ec2 describe-instances --query "Reservations[*].Instances[*].[InstanceId,PrivateIpAddress]"
```

Connect with all master and worker nodes using your private key.
## 2. Kubeadm Installation on EC2 Instances
Master and Worker Nodes (Both)

### **2.1. Update Packages**
```bash
sudo apt update && sudo apt upgrade -y
```

### **2.2. Disable Swap**
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### **2.3. Install Kubectl**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### **2.4. Configure Kernel Parameters**
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

### **2.5. Install CRIO Runtime**
```bash
sudo apt-get install -y cri-o
sudo systemctl enable --now crio
```

### **2.6. Install Kubernetes Components**
```bash
sudo apt-get install -y kubelet="1.29.0-*" kubectl="1.29.0-*" kubeadm="1.29.0-*"
sudo systemctl enable --now kubelet
```

### **2.7. Initialize the Kubernetes Master Node**
```bash
sudo kubeadm init
mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
```
Apply network plugin:
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
```
Generate a token for worker nodes:
```bash
kubeadm token create --print-join-command
```

### **2.8. Join Worker Nodes**
```bash
sudo kubeadm join <MasterPrivateIP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

## 3. Deploying Applications with Affinity and Anti-Affinity

### **Deploy Service A (Anti-Affinity)**
Create `service-a.yaml`:
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
```
Apply deployment:
```bash
kubectl apply -f service-a.yaml
```

### **Deploy Service B (Affinity)**
Create `service-b.yaml`:
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
curl http://service-a:80
curl http://service-b:80
```

### **Cleanup**
```bash
kubectl delete -f service-a.yaml
kubectl delete -f service-b.yaml
```

Your Kubernetes cluster is now fully set up with AWS, Kubeadm, and a deployment using node affinity and anti-affinity rules! 🚀