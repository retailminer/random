# Native Kubernetes Installation

Kubernetes is composed of master(s) and workers. The instructions below are for creating a bare-bones installation of a single master and several workers for __testing__ purposes. For a more complex, __production-grade__, Kubernetes installation, use tools such as Rancher Kubernetes Engine, or review [Kubernetes documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/){target=_blank} to learn how to customize the native installation.

## Prerequisites:

The script below assumes all machines have Ubuntu 20.04. For other Linux-based operating-systems see [Kubernetes documentation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/){target=_blank}. 


## Run on All Nodes

If not yet installed, install docker by performing the instructions [here](https://docs.docker.com/engine/install/ubuntu/){target=_blank}. Specifically, you can use a convenience script provided in the document:
``` shell
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

Change Docker to use `systemd` by editing 

```
sudo vi /etc/docker/daemon.json 
```

and adding:

``` JSON title="/etc/docker/daemon.json"
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```
For more information, see [container runtimes](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#docker){target=_blank}. Restart the docker service:

``` bash
sudo systemctl restart docker
```
## Run on Master Node

Install Kubernetes master:
``` shell
sudo sh -c 'cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF'

sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update
sudo apt-get install -y kubelet=1.23.5-00 kubeadm=1.23.5-00 kubectl=1.23.5-00
```

```
sudo swapoff -a
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --kubernetes-version=v1.23.5 --token-ttl 180h
```


The `kubeadm init` command above has emitted as output a `kubeadm join` command. Save it for joining the workers below. 

Copy the Kubernetes configuration file which provides access to the cluster: 
``` bash
mkdir .kube
sudo cp -i /etc/kubernetes/admin.conf .kube/config
sudo chown $(id -u):$(id -g) .kube/config
```

Add Kubernetes networking:
``` 
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Test that Kubernetes is up and running:
```
kubectl get nodes
```
Verify that the master node is ready

## Single Node Taint Removal

```
kubectl describe node <nodename> | grep Taints
```
You will get something like this (master or worker_node) 
```
Taints:             node-role.kubernetes.io/master:NoSchedule
```
To remove taint from node just run
```
kubectl taint node <nodename> node-role.kubernetes.io/master:NoSchedule-
```
## Run on Kubernetes Workers

For Kubernetes workers with GPU, you must install the NVIDIA prerequisites. We recommend using the NVIDIA GPU Operator __on top__ of Kubernetes. For further details see [NVIDIA prerequisites](cluster-install.md#step-2-nvidia)


=== "NVIDIA GPU Operator (recommended)"
    No additional work. Install the operator after Kubernetes is installed. 

=== "NVIDIA software on each node"
    On Worker Nodes with GPUs, install NVIDIA Docker and make it the default docker runtime as described [here](../cluster-prerequisites/#nvidia). Specifically, also add `systemd` by editing `/etc/docker/daemon.json` as follows:

    ``` JSON title="/etc/docker/daemon.json"
    {
        "exec-opts": ["native.cgroupdriver=systemd"],
        "default-runtime": "nvidia",
            "runtimes": {
            "nvidia": {
                "path": "nvidia-container-runtime",
                "runtimeArgs": []
            }
        }
    }
    ```

Install Kubernetes worker (any machine):
``` shell
sudo sh -c 'cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF'

sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet=1.23.5-00 kubeadm=1.23.5-00

sudo swapoff -a
```

Replace the following `join` command with the one saved from the init command above:

``` shell
sudo kubeadm join 10.0.0.3:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```

!!! Note
    The default token expires after 24 hours. If the token has expired, go to the master node and run `sudo kubeadm token create --print-join-command`. This will produce an up-to-date join command.


Return to the master node. Re-run `kubectl get nodes` and verify that the new node is ready.


## Permanently disable swap on all nodes

1. Edit the file /etc/fstab
2. Comment out any swap entry if such exists

## Avoiding Accidental Upgrades

To avoid accidental upgrade of Kubernetes binaries, it is recommended to _hold_ the version. Run the following on all nodes:

```
sudo apt-mark hold kubeadm kubelet kubectl
```

## Install NVIDIA GPU Operator

The preferred method to deploy the GPU Operator is using helm.

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 \
   && chmod 700 get_helm.sh \
   && ./get_helm.sh
```

```
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia \
   && helm repo update
```

Bare-metal/Passthrough with default configurations on Ubuntu¶
In this scenario, the default configuration options are used:

```
helm install --wait --generate-name \
     -n gpu-operator --create-namespace \
     nvidia/gpu-operator
```

Now make sure all pods are running in the kube-system kube-flannel and gpu-operator namespaces 

```
kubectl get pods -A
```
