
## **4. Fix hostnames first**

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

## **5. Configure&#x20;**`/etc/hosts`**&#x20;on all three machines**

Execute:

```
sudo nano /etc/hosts
```

Make sure these entries exist on all three:

```
127.0.0.1 localhost
127.0.1.1 kartikeyasoft

192.168.56.10 kartikeyasoft
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

Then we'll finish Worker2 and move to Kubernetes endpoint correction.
