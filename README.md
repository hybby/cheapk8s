# cheapk8s - Kubernetes The Cheap Way
## Overview
This repository serves as a reference to setting up a Kubernetes cluster on
your laptop suitable for using to complete the labs outlined in **Kubernetes
Fundamentals** (LFS258) by the Linux Foundation.

This is provided as a **Vagrantfile** and set of documentation that should
be easily reusable by anyone with a relatively recent Windows laptop.

I was frustrated when running through the labs that the only supported
configuration was to spin up cloud infrastructure, which would incur cost upon
the student outside of the existing course and examination fees.

I knew it should be possible to run locally, especially given the low-traffic 
and exploratory nature of the interactions it would be serving.

I'm a cheapskate, and wanted to play about with Kubernetes for the purposes of
certification without spending any of my own money. Don't worry, I'll still
get a round in.

## Getting Started
The happy-path is using a Windows laptop with Ubuntu running under WSL to run
Vagrant and Virtualbox. All commands are issued from within a WSL shell.

### System Requirements
I've assumed you have at least 8GiB of RAM and a hex-core processor.

We'll create 3 VMs:

  * Control Plane (2GiB, 2vCPU)
  * Worker 1 (2GiB, 2vCPU)
  * Worker 2 (2GiB, 2vCPU)

If you have less, adjust the Vagrantfile accordingly, but YMMV.

I've also assumed you don't have a very fancy network. Each VM is brought up
with one private interface (which all cluster comms happen over) and one NAT
interface to allow things like `apt-get` to work.

If you have different requirements, adjust the Vagrantfile accordingly. YMMV.

### Pre-requisites
These are things you need to have installed and configured before you can use
the things included in this repository to stand up a Kubernetes cluster.

1. Enable [WSL](https://learn.microsoft.com/en-us/windows/wsl/install) and install Ubuntu.
2. Install [VirtualBox](https://download.virtualbox.org) for Windows.
3. Install [Vagrant](https://www.vagrantup.com/) via WSL (`apt-get install vagrant`)
4. Configure [Vagrant to use WSL](https://www.vagrantup.com/docs/other/wsl)
5. Configure your `~/.bashrc` or `~/.zshrc` with `VAGRANT_WSL_ENABLE_WINDOWS_ACCESS="1"` to allow WSL Vagrant to work with Windows VirtualBox
6. Configure your `PATH` in `~/.bashrc` or `~/.zshrc` to include the Virtualbox binaries directory (e.g. `PATH="$PATH:/mnt/c/Program Files/Oracle/VirtualBox"`) for easy access to `VBox*.exe` binaries
7. Make sure you're not connected to a work VPN or similar.

You should be good to go now!

### Launching the VMs
From this directory, simply run `vagrant up`.

This will create 3 completely bare Ubuntu 20.04 servers that you can access
with `vagrant ssh <cp|wrk1|wrk2>`, etc.

It'll take a little while (15 minutes or so) as launching each box patches
some operating system packages and salso installs the VBoxGuestAdditions.

Now you can move onto the steps for configuring the control planes and workers.

I've kept these relatively verbose. It's useful to know what the commands do.

Follow along with LFS258 Lab 3 if you like.

## Configuring the Control Plane
Log into the Control Plane virtual machine.

```
vagrant ssh cp
```

Install any outstanding patches or OS updates.

```
sudo apt-get update && sudo apt-get dist-upgrade -y
```

Before we install Kuberentes, let's get ahead of things by setting the Node IP
of this system to the private IP address. This will help us force Kubernetes
traffic over the private network rather than confusing things by using the NAT
interface.

```
# "Network interface should be consistent for the same box", he says with hope.
private_network_ip=$(ip addr show enp0s8 | grep 'inet ' | awk '{print $2}' | awk -F\/ '{print $1}')
echo KUBELET_EXTRA_ARGS="--node-ip $private_network_ip" | sudo tee /etc/default/kubelet
```

Install some software dependencies

```
sudo apt install curl apt-transport-https vim git wget gnupg2 \
software-properties-common apt-transport-https ca-certificates uidmap -y
```

Turn off swap.

```
sudo swapoff -a
```

Turn off the firewall and AppArmor. It's cool, we're learning.

```
sudo ufw disable
sudo ufw status

sudo systemctl stop apparmor
sudo systemctl disable apparmor
```

Ensure we've loaded certain kernel modules needed by Kubernetes.

```
sudo modprobe overlay
sudo modprobe br_netfilter
```

Configure and apply some kernel parameters needed by Kubernetes.

```
cat << EOF | sudo tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

Install ContainerD

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt install containerd -y
```

Install and pin the version of Kubernetes that the labs want (12.23.1).

```
echo "deb  http://apt.kubernetes.io/  kubernetes-xenial  main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt-get update

sudo apt-get install -y kubeadm=1.23.1-00 kubelet=1.23.1-00 kubectl=1.23.1-00
sudo apt-mark hold kubelet kubeadm kubectl
```

Get the manifest for Calico.

```
wget https://docs.projectcalico.org/manifests/calico.yaml
```

Ensure that the setting `CALICO_IPV4POOL_CIDR` does not clash with IP ranges we're using.

```
grep -A1 CALICO_IPV4POOL_CIDR calico.yaml
```

By default this is set to 192.168.0.0/16, which is fine. 

This is the range containers on our cluster will get.

Now, for convenience configure some `/etc/hosts` entries for our other cluster nodes.

```
echo 10.0.0.175 k8scp k8scp.example.com | sudo tee -a /etc/hosts
echo 10.0.0.176 wrk1 wrk1.example.com | sudo tee -a /etc/hosts
echo 10.0.0.177 wrk2 wrk2.example.com | sudo tee -a /etc/hosts
```

Now create a Kubernetes cluster configuration. Custom as we've got multiple IPs

```
cat << EOF | sudo tee kubeadm-config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: 1.23.1
controlPlaneEndpoint: "k8scp:6443"
networking:
  podSubnet: $(grep -A1 CALICO_IPV4POOL_CIDR calico.yaml | grep value | awk '{print $NF}' | tr -d \")
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: $(ip addr show enp0s8 | grep 'inet ' | awk '{print $2}' | awk -F\/ '{print $1}')
  bindPort: 6443
EOF
```

Initialise the control plane.

```
sudo kubeadm init --config=kubeadm-config.yaml \
    --upload-certs \
    | sudo tee kubeadm-init.out
```

Configure `kubectl`.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Apply the Calico network config to the cluster:

```
kubectl apply -f calico.yaml
```

Configure `kubectl` bash completion

```
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> $HOME/.bashrc
```

Running `kubectl` commands should now work in general, and we're done.

```
kubectl get nodes
```

## Configuring Worker Nodes
You can either do these one at a time or both at once for expedience.

Some of this will look familiar. Some of it will need you to get stuff from
the Kubernetes control plane node we just set up, which I'll **highlight**.

First, log into the worker node.

```
vagrant ssh wrk1
```

Install any outstanding patches or OS updates.

```
sudo apt-get update && sudo apt-get dist-upgrade -y
```

Before we install Kuberentes, let's get ahead of things by setting the Node IP
of this system to the private IP address. This will help us force Kubernetes
traffic over the private network rather than confusing things by using the NAT
interface.

```
# "Network interface should be consistent for the same box", he says with hope.
private_network_ip=$(ip addr show enp0s8 | grep 'inet ' | awk '{print $2}' | awk -F\/ '{print $1}')
echo KUBELET_EXTRA_ARGS="--node-ip $private_network_ip" | sudo tee /etc/default/kubelet
```

Install some software dependencies

```
sudo apt install curl apt-transport-https vim git wget gnupg2 \
software-properties-common apt-transport-https ca-certificates uidmap -y
```

Turn off swap.

```
sudo swapoff -a
```

Turn off the firewall and AppArmor. It's cool, we're learning.

```
sudo ufw disable
sudo ufw status

sudo systemctl stop apparmor 
sudo systemctl disable apparmor
```

Ensure we've loaded certain kernel modules needed by Kubernetes.

```
sudo modprobe overlay
sudo modprobe br_netfilter
```

Configure and apply some kernel parameters needed by Kubernetes.

```
cat << EOF | sudo tee /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

Install ContainerD

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt install containerd -y
```

Install and pin the version of Kubernetes that the labs want (12.23.1).

```
echo "deb  http://apt.kubernetes.io/  kubernetes-xenial  main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt-get update

sudo apt-get install -y kubeadm=1.23.1-00 kubelet=1.23.1-00 kubectl=1.23.1-00
sudo apt-mark hold kubelet kubeadm kubectl
```

For convenience, configure some DNS using `/etc/hosts`:

```
echo 10.0.0.175 k8scp k8scp.example.com | sudo tee -a /etc/hosts
echo 10.0.0.176 wrk1 wrk1.example.com | sudo tee -a /etc/hosts
echo 10.0.0.177 wrk2 wrk2.example.com | sudo tee -a /etc/hosts
```

Now, **on the Control Plane node, get a 2hr duration token we can use to join the cluster**

```
# --------------------
# Run on Control Plane
# --------------------

# output of below looks like: 854upa.bjbcbgq0fvsxzz18
kubeadm token create

# output of below looks like: efe13a93950380e5f0da84af6369643e2d8be22b75746d7d5c922e22020022e9
openssl x509 -pubkey \
-in /etc/kubernetes/pki/ca.crt | openssl rsa \
-pubin -outform der 2>/dev/null | openssl dgst \
-sha256 -hex | awk '{print $NF}'
```

Take note of the output you get. You'll need it shortly.

Now, **on the Worker Node** you can join the cluster:

```
# ------------------
# Run on Worker Node
# ------------------
TOKEN="854upa.bjbcbgq0fvsxzz18"
DIGEST="efe13a93950380e5f0da84af6369643e2d8be22b75746d7d5c922e22020022e9"

sudo kubeadm join \
  --token "${TOKEN}" \
  k8scp:6443 \
  --discovery-token-ca-cert-hash "sha256:${DIGEST}"
```

Now, **on the Control Plane** you should see the worker node has joined and is Ready.

It will take about 2 minues to transition from `NotReady` to `Ready`.

Now's a good time to make sure that second worker node is also configured correctly.

You should now have a fully configured Kubernetes cluster. Hooray!


## Deployments
The world is your oyster now, but here's a sample "Hello, World!" you can use
to confirm everything's working as you expect.

Run all of these **from the Control Plane** node.

```
kubectl create deployment nginx --image=nginx
```

You should see the deployment go to Running:

```
vagrant@cp:~$ kubectl get pod -o wide
NAME                     READY   STATUS              RESTARTS   AGE    IP              NODE   NOMINATED NODE   READINESS GATES
nginx-85b98978db-dmtvl   0/1     ContainerCreating   0          100s   192.168.5.193   wrk1   <none>           <none>

...wait...

vagrant@cp:~$ kubectl get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE    IP              NODE   NOMINATED NODE   READINESS GATES
nginx-85b98978db-dmtvl   1/1     Running   0          100s   192.168.5.193   wrk1   <none>           <none>
```

And then hitting that IP should show return a webpage.

```
vagrant@cp:~$ curl 192.168.5.193
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

Hooray!
