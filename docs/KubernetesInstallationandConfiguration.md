# Introduction
This module is from the Pluralsight course Kubernetes Installation and Configuration for the Kuberenetes Administrator Path

# Setup
1. The setup for this Lab contains 4 VMs Ubuntu 18.04 or greater
2. 1 is a control plane node, 3 are worker nodes
3. /etc/hosts/ hosts file includes name to IP mappings for the VMs
4. Static IPs on individual VMs
5. Swap is disabled
6. Take snapshots prior to installation, this way you can install and revert to snapshot if needed

Login to the VMs using the command below:
'''
ssh azureuser@vmIPOrHostname -i <private-key>(c:\Users\asn\.ssh\KEYNAME.pem)
'''

Disable swap, swapoff on all nodes and then edit your /etc/fstab file for any swap partitions
You can recover the space with fdisk. You may want to reboot to ensure your config is ok

'''
swapoff -a
vi /etc/fstab
'''

## Install Packages

### Deploying the containerd packages.
Prerequisite: First load two modules and configure them to load on boot
These are part of https://kubernetes.io/docs/setup/production-environment/container-runtimes/

'''
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
'''

Setup required sysctl params, these persist across reboots
'''
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
'''

Apply the sysctl params without reboot
'''
sudo sysctl --system
'''

Install containerd
'''
sudo apt-get update
sudo apt-get install -y containerd
'''

Create a containerd configuration file
'''
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
'''

Set the cgroup driver for the containerd to systemd which is required for the kubelet
For more information on this config file see:
https://github.com/containerd/cri/blob/master/docs/config.md
https://github.com/containerd/containerd/blob/master/docs/ops.md

At the end of this section
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    ...
Add these two lines, indentation matters
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup=true
'''
sudo vi /etc/containerd/config.toml
'''

Restart containerd with the new configuration
'''
sudo systemctl restart containerd
'''

### Install Kubernetes packages -kubeadm, kubelet and kubectl
Add Google's apt repository gpg key
'''
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
'''

Add the Kubernetes apt repository
'''
sudo bash -c 'cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF'
'''

Update the package list and use apt-cache policy to inspect versions available in the repository
'''
sudo apt-get update
apt-cache policy kubelet | head -n 20
'''

Install the required packages, if needed we can request a specific version.
Use this version because in a later course we will upgrade the cluster to a newer version
'''
VERSION=1.20.1-00
sudo apt-get install -y kubelet=$VERSION kubeadm=$VERSION kubectl=$VERSION
sudo apt-mark hold kubelet kubeadm kubectl containerd
'''
OR
To install the latest omit the version parameters
'''
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl containerd
'''

1 - systemd Units
Check the status of our kubelet and our container runtime, containerd.
The kubelet will enter a crashloop until a cluster is created or the node is joined to an existing cluster.
'''
sudo systemctl status kubelet.service
sudo systemctl status contaienrd.service
'''

Ensure both are set to start when the system starts up
'''
sudo systemctl enable kubelet.service
sudo systemctl enable containerd.service
'''

## Bootstrapping a Cluster with kubeadm
The process defined below is customizable using kubeadm cmd line options or a yaml file to create customizable clusters.
Further details can be found at https://kubernetes.io/docs/references/setup-tools/kubeadm/kubeadm-init on how to customize this.

Step 1:
Run the following command

'''
kubeadm init
'''

Step 2:
Pre-flight Checks
Executes checks to ensure right permissions, resource on the machine and checks for a compatible container runtime running

Step 3:
Creates a Certificate Authority
* Creates a self-signed Certificate Authority (CA)
* Can be part of an external PKI
* Secures cluster communication making API server communication https
* Authentication of users and cluster components: the key lies in /etc/kubernetes/pki
* Distributed to each node in the cluster using kubeadmin as nodes are added to a cluster

Further details can be found at https://kubernetes.io/docs/references/setup-tools/kubeadm/kubeadm-init on how to customize this.

Step 4:
Generates kubeconfig files
* Kubeconfig files are used to define how to connect to your cluster
* - Client Certificates
* - Cluster API Server network location
* /etc/kubernetes is where these kubeconfig files are located
* - admin.conf (kubernetes-admin) It is the administrator account. Super-user
* - kubelet.conf (kubernetes-admin) Is used to hel the Kubelet to find the API server and which client cert to use to communicate with the API server.
* - controller-manager.conf (kubernetes-admin) Same as kubelet
* - scheduler.conf (kubernetes-admin) Same as kubelet

Step 5:
Generates the static pod manifests which will then be monitored by kubelet and start up the pods listed
* Manifest describes a configuration. Describes the configuration for cluster control plane pods.
* - Located in /etc/kubernetes/manifests
* - kubeadm init generates the manifests for etcd, controller manager, API Server, Scheduler
* Watched by the kubelet started automatically when the system starts over time
* Enables the startup of the cluster... without the cluster

Step 6:
Wait for the control plane Pods to start up on the Node. Thus etcd, API Server, Controller Manager and scheduler will run on Pods

Step 7:
Taint the control plane node. This is so that no worker node pods are created on the control plane node and only system pods get generated here.

Step 8:
Generates a bootstrap token to be able to add nodes to a cluster in the future.

Step 9:
Starts any Add-On components: DNS and kube-proxy.


## Pod Networking
* Single un NATed IP address per Pod
* Direct Routing
* * Configure infrastructure to support IP reachability between Pods and Nodes. This may not be feasible in a cloud scenario
* Overlay networking (SDN). Single Layer 3 Network where all Pods can communicate on using techinques like tunneling and encapsulation of the IP packets.
* * Flannel - Layer 3 Virtual Network
* * Calico - L3 and policy based traffic management
* * Weave Net - multi-host network

For full details on Overlay Networking visit https://kubernetes.io/docs/concepts/cluster-administration/networking/


## Creating a Control Plane Node

'''
wget https://docs.projectcalico.org/manifests/calicao.yaml

kubeadm config print init-defaults | tee ClusterConfiguration.yaml

sudo kubeadm init \
 --config=Cluster-Configuration.yaml \
 --cri-socket /run/containerd/containerd.sock

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f calico.yaml
'''

## Adding a Node to a Cluster
- Install Packages to the node like containerd, kubectl, kubeadm, kubelet
- kubeadm join to join a node to a cluster for networking  and certificates for security
- Download cluster information
- Node submits a Certficate Signing Request (CSR) into an API server  for use by kubelet on the node
- CA signs the CSR automatically sign that certificate and the kubeadm join will download that certificate and store the file in /var/lib/kubelet/pki
- Configures the kubelet.conf (TLS bootstrapping)

'''
kubeadm join <APIServerip:port> \
    -- token <Bootstraptoken> \
    -- discovery-token-ca-cert-hash <CertHash to communicate with API server of control plane node. From the output of kubeadmin init>
'''

## DEMO Steps
- Create our kubernetes cluster, specify a pod network range matching that in calico.yaml. Only on the control plane node download the yaml files for the pod network.
'''
wget https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/tigera-operator.yaml
wget https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/custom-resources.yaml
'''
- Look inside calico.yaml and find the setting for the Pod Network IP address range CALICO_IPV4POOL_CIDR, adjust if needed for your infrastructure to ensure that the Pod network IP range does not overlap with other networks in our infrastructure.
'''
vi calico.yaml
'''
- Generate a default kubeadm init configuraiton file... this defines the settigns of the cluster being built. If you get a warning about how docker is not installed... this is OK to ignore and is a bug in kubeadm
  For more info on kubeconfig configuration files see:
  https://kubernetes.io/docs/reference/setup-tools/kubeadm-init/#config-file
'''
kubeadm config print init-defaults | tee ClusterConfiguration.yaml
'''
- Inside default configuration file, we need to change four things:
- - The IP endpoint for the API Server localAPIEndpoint.advertiseAddress
- - nodeRegistration.criSocket from docker to containerd
- - Set the cgroup driver for the kubelet to systemd, it's not set in htis file yet. the default is cgroupfs
- - Edit kubernetesVersion to match the version you installed in Package Installation for containerd
'''
sed -i 's/ advertiseAddress: 1.2.3.4/ advertiseAddress: 10.0.0.4/' ClusterConfiguration.yaml

sed -i 's/ criSocket: \/var\/run\/dockershim\.sock/ criSocket: \/run\/containerd\/containerd\.sock/' ClusterConfiguration.yaml

cat <<EOF | cat >> ClusterConfiguration.yaml
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF

vi ClusterConfiguration.yanml
'''
- Need to add CRI socket since there's a check for docker in the kubeadm init process, if you don't you'll get this error...
    error execution phase preflight: docker is required for container runtime: exec: "docker": executable file not found
'''
sudo kubeadm init \
    --config=ClusterConfiguration.yaml \
    --cri-socket /var/run/containerd/containerd.sock
'''
- Before moving on revire the output of the cluster creation process including the kubeadm init phases as described earlier in this document, the admin.conf setup and the node join command
- Configure our account on the Control Plane Node to have admin access to the API servre from a non-privilleged account.
  '''
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  '''
- Creating the pod Network by deploying the yaml file for you pod network.
  '''
   kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/tigera-operator.yaml
  kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/custom-resources.yaml
  '''
- Look for all the system pods and calico pods to change to Running. The DNS pod won't start (pending) until the Pod network is deployed and Running.
  '''
  kubectl get pods --all-namespaces
  kubectl get pods --all-namespaces -watch
  '''
- Check the status of the Node and it should be running
  '''
  kubectl get nodes
  '''
- Check out the systemd unit.. It's no longer crashlooping because it has static pods to start. Remember the kubelet starts the static pods, and thus the control plane pods
  '''
  sudo systemctl status kubelet.service
  '''
- Let's check out the static pod manifests on the control plane node
  '''
  ls /etc/kubernetes/manifests