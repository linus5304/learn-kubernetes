create 3 servers
2 worker nodes
1 control plane

Distribution
ubuntu 20.04 Focal Fossa LTS

size
medium

<!-- command to rename the server -->
sudo hostnamectl set-hostname k8s-control

<!-- hostfile setup with mapping for all the servers so that they can communicate using hostnames -->
sudo vi /etc/hosts

172.31.54.84 k8s-control
172.31.53.102 k8s-worker1
172.31.48.74 k8s-worker2

<!-- Install and configure containerd on all the 3 servers -->
<!-- 1. Enable kernel modules:  This will make sure that anytime the server starts the 2 modules must be enabled--> 
cat << EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

<!-- 2.  Here we immediately enable those modules -->
sudo modprobe overlay
sudo modprobe br_netfilter

<!-- 3. Setting up system related configs required for k8s networking to work as expected -->
cat << EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

<!-- 4. Apply the above cofigs immediately -->
sudo sysctl --system

<!-- 5. Install containerd -->
sudo apt-get update && sudo apt-get install -y containerd

<!-- 6. setup containerd configuration file -->
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd

<!-- 7.  installing the k8s packages -->
<!-- it requires swap to be disabled -->
sudo swapoff -a
sudo apt-get update && sudo apt-get install -y apt-transport-https curl

<!-- The remaining points should be considered from the documentation
https://v1-29.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
 -->
<!-- 8. setup package repo for k8s packages -->
<!-- Update the apt package index and install packages needed to use the Kubernetes apt repository: -->
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

<!-- Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL: -->

# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

<!-- Add the appropriate Kubernetes apt repository. Please note that this repository have packages only for Kubernetes 1.29; for other Kubernetes minor versions, you need to change the Kubernetes minor version in the URL to match your desired minor version (you should also check that you are reading the documentation for the version of Kubernetes that you plan to install). -->
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

<!-- Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version: -->
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

<!-- (Optional) Enable the kubelet service before running kubeadm: -->
sudo systemctl enable --now kubelet


<!-- 9. initialise cluster only in the k8s-control -->
sudo kubeadm init --pod-network-cidr 192.168.0.0/16

<!-- 10. configure networking plugin -->
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

<!-- 11. joining worker nodes to the cluster -->
kubeadm token create --print-join-command
kubeadm join 172.31.54.84:6443 --token v9f8kl.h8j8aik0p08dlxa3 --discovery-token-ca-cert-hash sha256:985d67aefdec3ef06e9a6797303014d91b43cce82d042b36dcf4652b971b56d3


kubeadm join 10.0.1.101:6443 --token 9x1qw4.b13lhdy2us8k0ffm --discovery-token-ca-cert-hash sha256:b45a4dfd186e859856ef8b544206e65ef233a1a86a6384628c03836f05b9c48
kubeadm join 10.0.1.101:6443 --token q0p8bh.v2tlzero2javlio4 --discovery-token-ca-cert-hash sha256:b45a4dfd186e859856ef8b544206e65ef233a1a86a6384628c03836f05b9c481


<!-- K8s API -->
<!-- the kubectl uses the k8s API -->
kubectl get pods -n kube-system

kubectl get --raw /api/v1/namespaces/kube-system/pods

<!-- my-service.yml -->
apiVersion: v1
kind: Service
metadata:
    name: my-service
spec:
    selector:
        app: my-app
    ports:
        - protocol: TCP
          port: 80
          targetPort: 80

<!-- nginx-pod.yml -->
apiVersion: 1
kind: Pod
metadata:
    name: nginx
spec:
    containers:
    - name: nginx
      image: nginx
      ports:
      - containerPort: 80

apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80