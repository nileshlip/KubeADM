Kubernetes Cluster Setup on AWS EC2
1. AWS EC2 Instances Configuration
Master Node: Ubuntu 20.04 / 22.04 LTS
Worker Nodes: Ubuntu 20.04 / 22.04 LTS
Ensure each instance has at least:
 - 2 CPUs
 - 2 GB RAM
 - 20 To 30 GB SSD
2. Security Group Rules
Configure the security group to allow the following inbound traffic:
Port/Protocol	Purpose
22/TCP (SSH)	SSH access
6443/TCP	Kubernetes API server
2379-2380/TCP	etcd communication (Master)
10250-10255/TCP	Kubelet API
179/TCP	Calico networking (BGP)
8472/UDP	VXLAN (Calico) traffic
ICMP	Ping (for diagnostics)
Step 1: SSH into the EC2 Instances
Use your private key to SSH into both the master and worker nodes:
```bash
ssh -i 'your-key.pem' ubuntu@<instance-public-ip>
```
Step 2: Disable Swap on All Nodes
Kubernetes does not support swap. Disable it temporarily and permanently:
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
Step 3: Install Docker on All Nodes
Install Docker as the container runtime:
```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
```
Verify the Docker installation:
```bash
docker --version
```
Step 4: Install Kubernetes Components (kubeadm, kubelet, kubectl)
On all nodes, install the required Kubernetes packages:
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
Install kubeadm, kubelet, and kubectl:
```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
Verify installation:
```bash
kubeadm version
kubectl version --client
```
Step 5: Initialize the Master Node
On the master node, run the following command to initialize the control plane:
```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=<master-private-ip>
```
Configure kubectl on the Master Node:
```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Verify the master node status:
```bash
kubectl get nodes
```
Step 6: Install a Pod Network (Calico)
Install Calico for networking between pods:
```bash
kubectl apply -f https://docs.projectcalico.org/v3.25/manifests/calico.yaml
```
Check the status of Calico pods:
```bash
kubectl get pods --all-namespaces
```
Step 7: Join Worker Nodes to the Cluster
On the master node, get the join command:
```bash
kubeadm token create --print-join-command
```
Run the join command on each worker node:
```bash
sudo kubeadm join <master-private-ip>:6443 --token <token> --discovery-token-ca-cert-hash <hash>
```
Step 8: Verify the Cluster
On the master node, check the status of all nodes:
```bash
kubectl get nodes
```
You should see both the master and worker nodes in the READY state.
Step 9: Deploy a Test Application
Deploy nginx on the cluster:
```bash
kubectl run nginx --image=nginx --port=80
```
Expose nginx as a service:
```bash
kubectl expose pod nginx --type=NodePort --port=80
```
Get the service details:
```bash
kubectl get svc nginx
```
Access the nginx service using the public IP of any worker node and the NodePort from the output above:
```plaintext
http://<worker-public-ip>:<NodePort>
```
Step 10: Kubernetes Commands for Pods, Scaling, Replicas, Services, and Deployments
Here are some useful commands:
- Create a deployment:
```bash
kubectl create deployment my-deployment --image=nginx --replicas=3
```
- Scale a deployment:
```bash
kubectl scale deployment my-deployment --replicas=5
```
- Get the list of pods:
```bash
kubectl get pods
```
- Get detailed information about a specific pod:
```bash
kubectl describe pod <pod-name>
```
- Create a service:
```bash
kubectl expose deployment my-deployment --type=NodePort --port=80
```
- Get service details:
```bash
kubectl get svc my-deployment
```
Step 11: Troubleshooting
If a node shows as NotReady:
1. Ensure the Calico network is correctly installed.
2. Verify Docker is running on all nodes:
   ```bash
   sudo systemctl status docker
   ```
3. Check logs on the worker node:
   ```bash
   journalctl -u kubelet -f
   ```
If you need to reset a node:
```bash
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes/ /var/lib/etcd /var/lib/kubelet /etc/cni /var/lib/cni
```
Step 12: Conclusion
You have successfully set up a Kubernetes cluster on AWS EC2 using kubeadm with Calico networking. You can now deploy and manage applications on your cluster.
