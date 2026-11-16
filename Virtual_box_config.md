
## **1. Fix hostnames first**

On Master, keep:

```
hostnamectl set-hostname kartikeyasoft
```

On Worker1:

```
sudo hostnamectl set-hostname worker1
```

On Worker2:

```
sudo hostnamectl set-hostname worker2
```

Check on each machine:

```
hostname
```

Expected:

```
Master:   kartikeyasoft
Worker1:  worker1
Worker2:  worker2
```

## **2. Configure&#x20;**`/etc/hosts`**&#x20;on all three machines**

Execute:

```
sudo nano /etc/hosts
```

Make sure these entries exist on all three:

```
127.0.0.1 localhost
127.0.1.1 master

192.168.56.10 master
192.168.56.11 worker1
192.168.56.12 worker2
```

Remove conflicting old hostname mappings, if present. Keep the existing IPv6 localhost entries.


---------------------------------------------------


## **1. Correct Master Netplan configuration**

First, you already created a backup. Good.

Edit:

```
sudo nano /etc/netplan/01-netcfg.yaml
```

Replace its contents with:

```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 192.168.56.10/24

    enp0s8:
      dhcp4: true
```

Save the file.

## **2. Disable cloud-init networking**

You created this file:

```
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

Ensure it contains exactly:

```
network: {config: disabled}
```

Now remove the generated Netplan configuration that conflicts with your static configuration:

```
sudo rm /etc/netplan/50-cloud-init.yaml
```

Keep the `.bak` file as a backup; it is not an active `.yaml` configuration.

## **3. Apply the configuration**

Run:

```
sudo netplan generate
```

Then inspect:

```
sudo netplan get
```

Expected:

```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:
      addresses:
        - 192.168.56.10/24
      dhcp4: false
    enp0s8:
      dhcp4: true
```

If correct:

```
sudo netplan apply
```


## **Expected final network configuration**

| VM      | Host-Only     | NAT  |
| ------- | ------------- | ---- |
| Master  | 192.168.56.10 | DHCP |
| Worker1 | 192.168.56.11 | DHCP |
| Worker2 | 192.168.56.12 | DHCP |

**One important precaution:** Do not modify the Kubernetes API Server certificate yet. First stabilize the network on all three VMs.

Please share the output of:

```
sudo netplan get
ip -4 addr show enp0s3
ip -4 addr show enp0s8
```


------------------------------------------------------------------------------------------------------



The `kubeadm init` command must be executed **on the Master VM only**.

However, don't run the plain command:

```
sudo kubeadm init
```

It may automatically select the NAT IP `10.0.3.15` again, which is exactly the issue we're trying to avoid.

We'll initialize using a configuration file so the Host-Only IP is selected permanently.

## **1. Important network correction**

Your Host-Only network is `192.168.56.0/24`. Therefore, we should not use `192.168.0.0/16` for the Pod CIDR because it overlaps the Host-Only network.

Let's use:

| Network         | CIDR              |
| --------------- | ----------------- |
| Host-Only       | `192.168.56.0/24` |
| Pod network     | `10.244.0.0/16`   |
| Service network | `10.96.0.0/12`    |

We'll configure Calico to use `10.244.0.0/16` too.

## **2. Configure containerd on Master**

First, generate the default containerd configuration:

```
sudo mkdir -p /etc/containerd

sudo containerd config default | \
sudo tee /etc/containerd/config.toml >/dev/null
```

Enable systemd cgroups:

```
sudo sed -i \
  's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml
```

Restart containerd:

```
sudo systemctl restart containerd
sudo systemctl enable containerd
```

Verify:

```
sudo grep -n 'SystemdCgroup' /etc/containerd/config.toml

sudo systemctl status containerd --no-pager
```

Expected:

```
SystemdCgroup = true
active (running)
```

## **3. Create the kubeadm configuration**

On Master:

```
sudo nano /root/kubeadm-config.yaml
```

Paste:

```
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration

kubernetesVersion: v1.36.6

controlPlaneEndpoint: "192.168.56.10:6443"

apiServer:
  certSANs:
    - "192.168.56.10"
    - "10.0.3.15"

networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
  dnsDomain: "cluster.local"

---
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration

localAPIEndpoint:
  advertiseAddress: "192.168.56.10"
  bindPort: 6443

nodeRegistration:
  criSocket: "unix:///run/containerd/containerd.sock"
```

## **4. Validate the configuration**

```
sudo kubeadm config validate \
  --config=/root/kubeadm-config.yaml
```

Then check the images:

```
sudo kubeadm config images list \
  --config=/root/kubeadm-config.yaml
```

## **5. Initialize Master**

Run only after old cluster reset is verified

```
sudo kubeadm init \
  --config=/root/kubeadm-config.yaml
```

The important settings are:

- API Server advertised address: `192.168.56.10`
- Permanent control-plane endpoint: `192.168.56.10:6443`
- Certificate SAN includes `192.168.56.10`
- Pod CIDR: `10.244.0.0/16`



Do not run kubeadm init again yet - Fix

We need to cleanly reset the old control plane first. Do not use `--ignore-preflight-errors`; those errors represent real conflicts.

## **Stage 1 — Reset the old Master cluster**

You have confirmed that the old cluster data can be deleted.

### **Step 1. Run kubeadm reset**

On Master:

```
sudo kubeadm reset -f \
  --cri-socket=unix:///run/containerd/containerd.sock
```

This should stop and remove the old kubeadm-managed control-plane components.

### **Step 2. Clean up old CNI configuration**

```
sudo rm -rf /etc/cni/net.d
sudo rm -rf /var/lib/cni
```

### **Step 3. Remove old etcd data**

Because you authorized a complete rebuild:

```
sudo rm -rf /var/lib/etcd
```

### **Step 4. Remove old kubeconfig files**

```
sudo rm -rf /root/.kube
sudo rm -f /etc/kubernetes/admin.conf
sudo rm -f /etc/kubernetes/super-admin.conf
```

Do not manually remove the entire `/etc/kubernetes/pki` directory yet. Let kubeadm manage its certificate cleanup.

### **Step 5. Check whether the old control plane has stopped**

```
sudo ss -lntp | grep -E ':6443|:10250|:10257|:10259|:2379|:2380'
```

Ideally, this should return no output.

If ports remain occupied, inspect the processes before doing anything else:

```
sudo crictl ps -a
```

If `crictl` is not installed, use:

```
sudo ctr -n k8s.io containers list
```

## **Stage 2 — Verify the Master is clean**

Run:

```
sudo kubeadm-cluster-preflight.sh
```

Expected:

| Check                        | Expected         |
| ---------------------------- | ---------------- |
| Master IP                    | `192.168.56.10`  |
| NAT IP                       | `10.0.3.15`      |
| containerd                   | Active           |
| `/etc/kubernetes/admin.conf` | Absent           |
| `/etc/kubernetes/manifests`  | No old manifests |
| `/var/lib/etcd`              | Empty            |
| Port 6443                    | Not listening    |

## **Stage 3 — Initialize the new cluster**

Once the reset is verified, use your already validated configuration:

```
sudo kubeadm init \
  --config=/root/kubeadm-config.yaml
```

The new control plane will use:

```
API Server: 192.168.56.10:6443
Advertise Address: 192.168.56.10
Pod CIDR: 10.244.0.0/16
Service CIDR: 10.96.0.0/12
```

After successful initialization, we will configure kubectl and install Calico.

Yes, this command installs **Calico CNI (Container Network Interface)**, which provides Pod-to-Pod networking and NetworkPolicy support in Kubernetes.

However, I recommend using **Calico v3.33.0 instead of v3.30.0** for your Kubernetes v1.36.6 cluster. The current Calico documentation tests v3.33 against Kubernetes 1.36.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.tigera.io\&sz=32)

Calico Documentation+1



### **1. Important: Run this only after successful&#x20;**`kubeadm init`

On your Master, first verify:

```
kubectl get nodes
kubectl get pods -n kube-system
```

If the API Server is running and kubectl works, proceed.

### **2. Install Calico**

For your new cluster, I recommend the documented Tigera Operator installation rather than the older standalone manifest.

Run on Master:

```
kubectl create -f \
https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/v3_projectcalico_org.yaml
```

```
kubectl create -f \
https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/tigera-operator.yaml
```

Download the custom resources:

```
curl -O \
https://raw.githubusercontent.com/projectcalico/calico/v3.33.0/manifests/custom-resources.yaml
```

Change the default Pod CIDR to match your kubeadm configuration:

```
sed -i 's/192.168.0.0\/16/10.244.0.0\/16/g' custom-resources.yaml
```

Verify:

```
grep -n 'cidr:' custom-resources.yaml
```

Expected:

```
cidr: 10.244.0.0/16
```

Apply:

```
kubectl create -f custom-resources.yaml
```

These steps follow Calico's documented operator installation approach.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.tigera.io\&sz=32)

Calico Documentation+1



### **3. Verify Calico**

```
kubectl get pods -n tigera-operator
kubectl get pods -n calico-system
kubectl get tigerastatus
kubectl get nodes -o wide
```

Wait until the Calico components are healthy and the Master becomes `Ready`.

Your current next step

Your earlier Master initialization failed because the old control-plane ports and etcd data were still present. The Worker3 join also reported a containerd CRI problem.

**Have you successfully completed a fresh&#x20;**`kubeadm init`**&#x20;on Master yet?**

Don't apply the Calico manifest or operator until the new Master API Server is initialized and healthy.



I found the issue. Your existing Calico configuration has:

```
nodeAddressAutodetectionV4:
  firstFound: true
```

When we added `interface: enp0s3`, both detection methods remained enabled. Calico allows only one autodetection method at a time.

We need to **replace&#x20;**`firstFound`**&#x20;with&#x20;**`interface`, not add another method.

## **Step 1: Correct the configuration**

Run this command on `master`:

```
kubectl patch installation default --type=json \
-p='[
  {"op":"remove","path":"/spec/calicoNetwork/nodeAddressAutodetectionV4/firstFound"},
  {"op":"add","path":"/spec/calicoNetwork/nodeAddressAutodetectionV4/interface","value":"enp0s3"}
]'
```

Expected response:

```
installation.operator.tigera.io/default patched
```

## **Step 2: Verify**

```
kubectl get installation default -o yaml
```

You should now see:

```
nodeAddressAutodetectionV4:
  interface: enp0s3
```

There should be no `firstFound: true`.

## **Step 3: Monitor Calico**

```
kubectl get pods -n calico-system -w
```

Once the pods settle, check:

```
kubectl get nodes -o wide
kubectl get tigerastatus
```



Expected result

&#x20;Calico should detect the unique Host-Only IP on each VM instead of the shared NAT IP `10.0.3.15`.


