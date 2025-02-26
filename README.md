# Kubernetes Cluster Setup on AWS EC2

## AWS EC2 Instances Configuration
- **Master Node**: Ubuntu 20.04 / 22.04 LTS  
- **Worker Nodes**: Ubuntu 20.04 / 22.04 LTS  
- **Minimum Requirements for Each Instance:**
  - 2 CPUs
  - 2 GB RAM
  - 20 to 30 GB SSD

## Security Group Rules
Configure the security group to allow the following inbound traffic:

| Port/Protocol  | Purpose |
|---------------|---------|
| 22/TCP       | SSH access |
| 6443/TCP     | Kubernetes API server |
| 2379-2380/TCP | etcd communication (Master) |
| 10250-10255/TCP | Kubelet API |
| 179/TCP      | Calico networking (BGP) |
| 8472/UDP     | VXLAN (Calico) traffic |
| ICMP         | Ping (for diagnostics) |

---

## Step 1: SSH into the EC2 Instances
Use your private key to SSH into both the master and worker nodes:
```sh
ssh -i 'your-key.pem' ubuntu@<instance-public-ip>
```

## Step 2: Disable Swap on All Nodes
Kubernetes does not support swap. Disable it temporarily and permanently:
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
Verify the Docker installation:
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
```
Check the status of Calico pods:
```sh
kubectl get pods --all-namespaces
```

## Step 7: Join Worker Nodes to the Cluster
On the master node, get the join command:
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
You should see both the master and worker nodes in the `READY` state.

## Step 9: Deploy a Test Application
```sh
kubectl run nginx --image=nginx --port=80
```
Expose nginx as a service:
```sh
kubectl expose pod nginx --type=NodePort --port=80
```
Get the service details:
```sh
kubectl get svc nginx
```
Access the nginx service:
```sh
http://<worker-public-ip>:<NodePort>
```

---

## Step 10: Kubernetes Commands for Pods, Scaling, Replicas, Services, and Deployments

### General Commands:
```sh
kubectl get all               # List all resources
kubectl get pod               # List all pods
kubectl run my-pod --image=redhat/ubi8 --port=80  # Create a Pod
```

### Image Management:
```sh
docker search redhat         # Search images from DockerHub
```

### Manifest Files:
```sh
kubectl get po -o yaml       # Get manifest file
kubectl apply -f pod1.yml    # Create a pod using a manifest file
```

### Node & Pod Information:
```sh
kubectl get po -o wide       # Get node information
kubectl describe po testpod  # Get details of a pod
```

### Backup YAML Content:
```sh
kubectl get po testpod -o yaml > mypod.yaml  # Backup your YAML content
```

### Container Access:
```sh
kubectl exec testpod -it -c denish -- /bin/bash  # Access container shell
```

### Replicas & Scaling:
```sh
nano replica.yaml                    # Create manifest file
kubectl get rs                        # List all ReplicaSets
kubectl scale rs myreplicaset --replicas=8  # Scale up/down ReplicaSets
```

### Deployments:
```sh
kubectl create -f deploy.yaml         # Create Deployment using a manifest
kubectl scale deployment nginx-deployment --replicas=5  # Scale up/down Deployment
```

### Deletion Commands:
```sh
kubectl delete rs myreplicaset   # Delete ReplicaSet
kubectl delete pod testpod       # Delete Pod
```

### Help Command:
```sh
kubectl –help                     # View all Kubernetes commands
```

---

## Step 11: Troubleshooting
If a node shows as `NotReady`:
- Ensure the Calico network is correctly installed.
- Verify Docker is running on all nodes:
  ```sh
  sudo systemctl status docker
  ```
- Check logs on the worker node:
  ```sh
  journalctl -u kubelet -f
  ```

If you need to reset a node:
```sh
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes/ /var/lib/etcd /var/lib/kubelet /etc/cni /var/lib/cni
```

---

## Step 12: Conclusion
You have successfully set up a Kubernetes cluster on AWS EC2 using `kubeadm` with Calico networking. You can now deploy and manage applications on your cluster.

