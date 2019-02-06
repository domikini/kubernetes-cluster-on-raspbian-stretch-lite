## Install Docker

This installs specific version of Docker

`$ curl -sSL get.docker.com | VERSION=18.06.1-ce sh && \
sudo usermod <user> -aG docker
newgrp docker`

## Disable swap and reboot

For Kubernetes 1.7 and onwards you will get an error if swap space is enabled.
Turn off swap:

`$ sudo dphys-swapfile swapoff && sudo dphys-swapfile uninstall && sudo update-rc.d dphys-swapfile remove`
  
This should now show no entries:

`$ sudo swapon --summary`


Edit /boot/cmdline.txt. Add this text at the end of the line, but don't create any new lines:

`cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory`

Now reboot - do not skip this step.


## Install Kubernetes tools

Add repo lists & install kubeadm

`$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - && echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list && sudo apt-get update -q && sudo apt-get install -qy kubeadm`


## Initialize your master node

You now have two new commands installed:

- kubeadm - used to create new clusters or join an existing one

- kubectl - the CLI administration tool for Kubernetes

### Pre-pull images

kubeadm now has a command to pre-pull the requisites Docker images needed to run a Kubernetes master, type in:

`$ sudo kubeadm config images pull -v3`


### If using Weave Net

Initialize your master node:

`$ sudo kubeadm init --token-ttl=0`


## Install Weave Net Driver

This installs Weave Net driver which is used for the cluster

`$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`


## Join masternode from worker nodes
To join master node from worker nodes, run command:

`$ kubeadm join 192.168.0.108:6443 --token <token_number> --discovery-token-ca-cert-hash sha256:<sha256_hash_number>`

## Restart Kubernetes on master
In order to restart Kubernetes. Run command:

`$ kubeadm reset -f`

Remove the folder /var/lib/kubelet

`$ rm -Rf /var/lib/kubelet`


