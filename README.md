
# Kubernetes Cluster Setup on AWS EC2

## AWS EC2 Instances Configuration
**Master Node:** Ubuntu 20.04 / 22.04 LTS  
**Worker Nodes:** Ubuntu 20.04 / 22.04 LTS  
Ensure each instance has at least:
- 2 CPUs
- 2 GB RAM
- 20 to 30 GB SSD

## Security Group Rules
Configure the security group to allow the following inbound traffic:

| Port/Protocol | Purpose |
|--------------|---------|
| 22/TCP (SSH) | SSH access |
| 6443/TCP | Kubernetes API server |
| 2379-2380/TCP | etcd communication (Master) |
| 10250-10255/TCP | Kubelet API |
| 179/TCP | Calico networking (BGP) |
| 8472/UDP | VXLAN (Calico) traffic |
| ICMP | Ping (for diagnostics) |

## Step 1: SSH into the EC2 Instances
Use your private key to SSH into both the master and worker nodes:
```sh
ssh -i 'your-key.pem' ubuntu@<instance-public-ip>
```

## Step 2: Disable Swap on All Nodes
```sh
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

## Step 3: Install Docker on All Nodes
```sh
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
```
Verify Docker installation:
```sh
docker --version
```

## Step 4: Install Kubernetes Components (kubeadm, kubelet, kubectl)
```sh
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Install kubeadm, kubelet, and kubectl:
```sh
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
Verify installation:
```sh
kubeadm version
kubectl version --client
```

## Step 5: Initialize the Master Node
```sh
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=<master-private-ip>
```
Configure kubectl on the Master Node:
```sh
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Verify the master node status:
```sh
kubectl get nodes
```

## Step 6: Install a Pod Network (Calico)
```sh
kubectl apply -f https://docs.projectcalico.org/v3.25/manifests/calico.yaml
kubectl get pods --all-namespaces
```

## Step 7: Join Worker Nodes to the Cluster
```sh
kubeadm token create --print-join-command
```
Run the join command on each worker node:
```sh
sudo kubeadm join <master-private-ip>:6443 --token <token> --discovery-token-ca-cert-hash <hash>
```

## Step 8: Verify the Cluster
```sh
kubectl get nodes
```

## Step 9: Deploy a Test Application
```sh
kubectl run nginx --image=nginx --port=80
kubectl expose pod nginx --type=NodePort --port=80
kubectl get svc nginx
```
Access the nginx service using the public IP of any worker node and the NodePort:
```sh
http://<worker-public-ip>:<NodePort>
```

## Step 10: Kubernetes Commands for Pods, Scaling, Replicas, Services, and Deployments

### Basic Kubernetes Commands
```sh
kubectl get all  # List all resources
kubectl get pod  # List all pods
kubectl get po -o yaml  # Get manifest file
kubectl get po -o wide  # Get node information
kubectl describe po testpod  # Get details of a pod
kubectl get po testpod -o yaml > mypod.yaml  # Backup YAML content
kubectl exec testpod -it -c denish -- /bin/bash  # Enter a container
kubectl delete pod testpod  # Delete a pod
kubectl –help  # See all K8S commands
kubectl get rs  # List of ReplicaSets
kubectl scale rs myreplicaset --replicas=8  # Scale ReplicaSets
kubectl create -f deploy.yaml  # Create deployment using YAML
kubectl scale deployment nginx-deployment --replicas=5  # Scale deployment
kubectl run my-replicaset --image=ubuntu --replicas=3  # Create ReplicaSet manually
kubectl create deployment mydeployment --image=ubuntu --replicas=2  # Create Deployment manually
kubectl scale deployment mydeployment --replicas=5  # Scale deployment
```

### Creating a Pod
```sh
vim pod1.yml
---
apiVersion: v1
kind: Pod
metadata:
  name: testpod
spec:
  containers:
  - name: denish
    image: ubuntu
    command: ["/bin/bash", "-c", "while true; do echo Hello-devops_engineer; sleep 5 ; done"]
  restartPolicy: Never
```
Apply the pod:
```sh
kubectl apply -f pod1.yml
```

### Creating a Multi-Container Pod
```sh
vim pod2.yml
---
apiVersion: v1
kind: Pod
metadata:
  name: testpod3
spec:
  containers:
  - name: nilesh
    image: ubuntu
    command: ["/bin/bash", "-c", "while true; do echo sevenmentor pune; sleep 5 ; done"]
  - name: huzaifa
    image: ubuntu
    command: ["/bin/bash", "-c", "while true; do echo Hello-devops engineers; sleep 5 ; done"]
```
Apply the pod:
```sh
kubectl apply -f pod2.yml
```

### ReplicaSet Example
```sh
vim myreplicaset.yml
---
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myreplicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: mycontainer
        image: ubuntu
        command: ["/bin/bash", "-c", "while true; do echo Running ReplicaSet; sleep 5; done"]
```
Apply the ReplicaSet:
```sh
kubectl apply -f myreplicaset.yml
```

## Creating and Running ReplicaSets and Deployments Manually

### Creating a ReplicaSet Manually
You can create a ReplicaSet directly using the `kubectl` command without needing a YAML file. Here’s how:

```sh
kubectl create rs myreplicaset --replicas=3 --image=ubuntu --command -- /bin/bash -c "while true; do echo Running ReplicaSet; sleep 5; done"
```

### Scaling the ReplicaSet
To scale the ReplicaSet, you can use the following command:

```sh
kubectl scale rs myreplicaset --replicas=5
```

### Creating a Deployment Manually
You can also create a Deployment directly using the `kubectl` command:

```sh
kubectl create deployment mydeployment --image=ubuntu --replicas=2 -- /bin/bash -c "while true; do echo Running Deployment; sleep 5; done"
```

### Scaling the Deployment
To scale the Deployment, use this command:

```sh
kubectl scale deployment mydeployment --replicas=4
```

### Verifying the Created Resources
To verify that your ReplicaSet and Deployment are running, you can use:

```sh
kubectl get rs
kubectl get deployments
```

### Conclusion
You have successfully set up a Kubernetes cluster on AWS EC2 using kubeadm with Calico networking. You can now deploy and manage applications on your cluster.
