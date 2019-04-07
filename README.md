## Change hostname
Use the raspi-config utility to change the hostname to k8s-master-1 or similar and then reboot.

## Set a static IP address
It's not fun when your cluster breaks because the IP of your master changed. The master's certificates will be bound to the IP address, so let's fix that problem ahead of time:

cat >> /etc/dhcpcd.conf
Paste this block:

```
interface eth0
static ip_address=192.168.0.100/24  
static routers=192.168.0.1  
static domain_name_servers=8.8.8.8  
```

## Install Docker

This installs specific version of Docker

```
$ curl -sSL get.docker.com | VERSION=18.06.1-ce sh
$ sudo usermod <user> -aG docker  
$ newgrp docker  
```

## Disable swap and reboot

For Kubernetes 1.7 and onwards you will get an error if swap space is enabled.
Turn off swap:

`$ sudo dphys-swapfile swapoff && sudo dphys-swapfile uninstall && sudo update-rc.d dphys-swapfile remove`
  
This should now show no entries:

`$ sudo swapon --summary`


Edit /boot/cmdline.txt. Add this text at the end of the line, but don't create any new lines:

`cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory`

Now reboot - do not skip this step.


## Install Kubernetes tools with specific version of Kubernetes

Add repo lists & install kubeadm

`$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - && echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list && sudo apt-get update -q && sudo apt-get install -qy kubelet=1.12.7-00 kubectl=1.12.7-00 kubeadm=1.12.7-00`


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

### Configure path to configuration file

After running init command. Run the following commands in order to setup configuration file and put it in a folder in the user home folder. Set the correct permissions for the file.

 `mkdir -p $HOME/.kube`  
 `sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`  
 `sudo chown $(id -u):$(id -g) $HOME/.kube/config`  


## Install Weave Net Driver

This installs Weave Net driver which is used for the cluster

`$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`


## Join masternode from worker nodes
To join master node from worker nodes, run command:

`$ kubeadm join <ip_number>:<port> --token <token_number> --discovery-token-ca-cert-hash sha256:<sha256_hash_number>`

## Restart Kubernetes on master
In order to restart Kubernetes. Run command:

`$ kubeadm reset -f`

The -f flag let you execute reset without a confirmation prompt.

Remove the folder /var/lib/kubelet

`$ rm -Rf /var/lib/kubelet`

## Install Kubernetes dashboard on master node
`kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml`


## Test Kubernetes cluster by deploying test application
Test the Kubernetes cluster by deploying the following Markdownrender application.

Create a file and name it function.yml.
`touch function.yml`

Paste the following code inside the function.yml file.
```
apiVersion: v1
kind: Service
metadata:
  name: markdownrender
  labels:
    app: markdownrender
spec:
  type: NodePort
  ports:
    - port: 8080
      protocol: TCP
      targetPort: 8080
      nodePort: 31118
  selector:
    app: markdownrender
---
apiVersion: apps/v1beta1 # for versions before 1.6.0 use extensions/v1beta1
kind: Deployment
metadata:
  name: markdownrender
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: markdownrender
    spec:
      containers:
      - name: markdownrender
        image: functions/markdownrender:latest-armhf
        imagePullPolicy: Always
        ports:
        - containerPort:
```

Run and create the deployment.
`kubectl create -f function.yml`

Check that the deployment is successful.
`kubectl get deployments`



