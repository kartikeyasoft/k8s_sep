# Install Kubernetes Cluster using kubeadm
Follow this documentation to set up a Kubernetes cluster on __Ubuntu__.

This documentation guides you in setting up a cluster with one master node and one worker node.

## { Master - Worker } Install Docker on 2 Nodes.

#### Update the apt package index and install packages to allow apt to use a repository over HTTPS:
```
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

```
#### Install the Docker packages.
```
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## { Master - Worker } install kubeadm, kubelet, and kubectl on 2 Nodes.

#### Update the apt package index and install packages needed to use the Kubernetes apt repository:
```
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

```
#### Download the Google Cloud public signing key:
```
# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.36/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

```

#### Add the Kubernetes apt repository:
```
# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.36/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

```

#### Update apt package index, install kubelet, kubeadm and kubectl, and pin their version:
```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

```
#### Enable the kubelet service before running kubeadm:
```
sudo systemctl enable --now kubelet
```

#### Restart docker and kubelet

```
systemctl restart kubelet
systemctl daemon-reload

```

#### Troubleshooting steps

```
cd /etc/containerd
rm -rf *;
systemctl restart containerd

```

#### Kubernetes Master Node Configuration - { Master Node Config }

```
sudo kubeadm init

```

#### kubeconfig update - { Master Node Config }
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```
#### Worker Node Join Token - { Master Node Config }
```
sudo kubeadm token create --print-join-command

```

#### K8S Cluster CNI Configuration: - { Master Node Config }
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

```
```
swapoff -a
kubeadm reset --force
```

#### Verify nodes status  - { Master Node Config }
```
kubectl get nodes

```


1. **Backup existing data**:
   ```bash
   sudo mv /var/lib/etcd /var/lib/etcd.backup
   ```

2. **Create fresh directory**:
   ```bash
   sudo mkdir /var/lib/etcd
   ```

3. **Set proper permissions**:
   ```bash
   sudo chmod 700 /var/lib/etcd
   ```

4. **Initialize cluster**:
   ```bash
   sudo kubeadm init --ignore-preflight-errors=DirAvailable--var-lib-etcd
   ```
