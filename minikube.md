https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fbinary+download

sudo apt update
sudo apt install -y curl wget apt-transport-https ca-certificates gnupg lsb-release

sudo apt install -y docker.io
sudo systemctl enable docker --now
sudo usermod -aG docker $USER

curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

minikube version

curl -LO "https://dl.k8s.io/release/$(curl -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

minikube start --driver=docker

minikube dashboard

kubectl get nodes

kubectl create deployment hello-minikube --image=kicbase/echo-server:1.0
kubectl expose deployment hello-minikube --type=NodePort --port=8080

kubectl get service
minikube service hello-minikube
kubectl port-forward service/hello-minikube 7080:8080

http://localhost:7080

