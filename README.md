# Monitoring a Kubernetes Cluster with Prometheus and Grafana on AWS EC2

A voting app deployed on a kind (Kubernetes in Docker) cluster running on an AWS EC2 Ubuntu instance, monitored with Prometheus and Grafana installed through Helm (kube-prometheus-stack).

## Architecture
- AWS EC2 (Ubuntu, c7i-flex.large, 4 GB RAM, 30 GB disk)
- kind cluster: 1 control-plane node and 1 worker node
- Voting app (vote, redis, worker, db, result) in the default namespace
- Prometheus, Grafana, Alertmanager and exporters in the monitoring namespace

## Prerequisites
- Security group inbound rules: 22 (SSH), 9090 (Prometheus), 3000 (Grafana), 5000 (vote app), source set to my IP only

## Setup

1. Install Docker
```
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER
newgrp docker
```

2. Clone the voting app and create the cluster
```
git clone https://github.com/LondheShubham153/k8s-kind-voting-app.git
cd k8s-kind-voting-app/kind-cluster
chmod +x install_kind.sh install_kubectl.sh
./install_kind.sh
./install_kubectl.sh
kind create cluster --config=config.yml --name=my-cluster
kubectl get nodes
```

3. Deploy the voting app
```
cd ../k8s-specifications
kubectl apply -f .
kubectl get pods
```

4. Install Helm
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

5. Install Prometheus and Grafana
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install kind-prometheus prometheus-community/kube-prometheus-stack --namespace monitoring
kubectl get pods -n monitoring
```

6. Expose the services
```
nohup kubectl port-forward svc/kind-prometheus-kube-prome-prometheus -n monitoring 9090:9090 --address=0.0.0.0 > prom.log 2>&1 &
nohup kubectl port-forward svc/kind-prometheus-grafana -n monitoring 3000:80 --address=0.0.0.0 > grafana.log 2>&1 &
nohup kubectl port-forward svc/vote 5000:5000 --address=0.0.0.0 > vote.log 2>&1 &
```

- Prometheus: http://EC2-PUBLIC-IP:9090
- Grafana: http://EC2-PUBLIC-IP:3000 (user: admin)
- Vote app: http://EC2-PUBLIC-IP:5000

## Monitoring
Example Prometheus queries:
```
sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) by (pod)
sum(container_memory_working_set_bytes{namespace="default"}) by (pod)
sum(rate(container_network_receive_bytes_total{namespace="default"}[5m])) by (pod)
kube_pod_status_phase{namespace="default"}
```

Grafana dashboard: Dashboards, New, Import, enter ID 15760, choose the Prometheus data source, then Import.

## Problems I faced and how I fixed them
- A 1 GB t3.micro instance could not run the cluster. Fixed by moving to a 4 GB instance (c7i-flex.large).
- Two worker nodes failed to join (kubelet timeout). Fixed by reducing the cluster to one worker.
- The 10 GB disk was too small. Fixed by growing the volume to 30 GB, then running growpart and resize2fs.
- Wrong package name: the Docker engine on Ubuntu is docker.io, not docker.
- Port-forward local port must match the port used in the browser (Grafana 3000:80).

## Cleanup
```
kind delete cluster --name my-cluster
```
Then stop or terminate the EC2 instance to avoid charges.

## Credit
Based on the k8s-kind-voting-app project by Shubham Londhe.
