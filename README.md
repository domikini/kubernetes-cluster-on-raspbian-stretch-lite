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
$ curl -sSL get.docker.com | VERSION=18.06.3-ce sh
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

`$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - && echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list && sudo apt-get update -q && sudo apt-get install -qy kubelet kubectl kubeadm`


If you want to install a specific version of Kubernetes, use the following command instead:

`$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - && echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list && sudo apt-get update -q && sudo apt-get install -qy kubelet=1.12.7-00 kubectl=1.12.7-00 kubeadm=1.12.7-00`


## Initialize your master node

You now have two new commands installed:

- kubeadm - used to create new clusters or join an existing one

- kubectl - the CLI administration tool for Kubernetes

### Pre-pull images

kubeadm now has a command to pre-pull the requisites Docker images needed to run a Kubernetes master, type in:

`$ sudo kubeadm config images pull --kubernetes-version v1.12.5`

(Check version by checking kubeadm version. Type `kubeadm version`)


### If using Weave Net

Initialize your master node:

`$ sudo kubeadm init --token-ttl=0`

If you want to run a specific version of Kubernetes

`$ sudo kubeadm init --token-ttl=0 --kubernetes-version 1.15.6`

### If using Flannel:

`$ sudo kubeadm init --token-ttl=0 --pod-network-cidr=10.244.0.0/16`

### Troubleshoot init phase
#### If you need to restart init
Run kubeadm init command with the --ignore-preflight-errors=all option

#### If you get cgroup driver warning
If you get warning (Switch cgroup driver, from cgroupfs to systemd) when running Kubernetes. Input the following in terminal:

```
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

Also, do not forget to open the firewall port e.g. 6443 for Kubernetes.

`sudo ufw allow 6443`

Init step can take a long time, up to 15 minutes.

Sometimes this stage can fail, if it does then you should patch the API Server to allow for a higher failure threshold during initialization around the time you see.

`sudo sed -i 's/failureThreshold: 8/failureThreshold: 20/g' /etc/kubernetes/manifests/kube-apiserver.yaml`

### Configure path to configuration file

After running init command. Run the following commands in order to setup configuration file and put it in a folder in the user home folder. Set the correct permissions for the file.

 `mkdir -p $HOME/.kube`  
 `sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config`  
 `sudo chown $(id -u):$(id -g) $HOME/.kube/config`  


## Install Weave Net Driver

This installs Weave Net driver which is used for the cluster

`$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`

If the Kuberentes version is not supported by the latest Weave Net driver, the above execution will fail. If that is the case, you may want to use an older version of Weave Net driver. In order to do that, download the yaml file from above link.

`wget "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`

Rename the output file to e.g. weave.yaml. Edit the file with and editor. Find the entries where weave-npc and weave-kube is specified and change the version number to the one you want to execute. After editing, save the changes and execute the yaml file.

`kubectl apply -f weave.yaml`

In my case, for kubeadmin version 1.15.12, Weave Net driver 2.6.4 worked well.

## Install Flannel (alternative)

Install the Flannel driver on the master:

`$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/c5d10c8/Documentation/kube-flannel.yml`

Check if your flannel version is compatible with the Kubernetes version you have installed. Follow the Flannel instruction which is applicable for your Kubernetes version.
https://github.com/coreos/flannel/blob/master/Documentation/kubernetes.md

## Join masternode from worker nodes
To join master node from worker nodes, run command:

`$ kubeadm join <ip_number>:<port> --token <token_number> --discovery-token-ca-cert-hash sha256:<sha256_hash_number>`

To create more tokens for joining:

`kubeadm token create --print-join-command`

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

## How to connect to private docker registry with Kubernetes secret
In order to pull from a private Docker registry from Kubernetes, a secret with the Docker credentials must be created.

First log in to Docker registry:
`docker login <url-to-Docker-registry:port>`

The credentials are stored in the path ~/.docker.config.json

Create Kubernetes secret by using the command:
`kubectl create secret generic regcred --from-file=.dockerconfigjson=<absolute-path-to-docker-config.json-file> --type=kubernetes.io/dockerconfigjson`

Now you can use the secret by specifying it in the deployment file in the template section e.g:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
  labels:
    app: example-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: example-app
  template:
    metadata:
      labels:
        app: example-app
      spec:
        containers:
        - name: example-app
          image: example.server.com:8080/example-app:latest
          ports:
          - containerPort: 8090
        imagePullSecrets:
        - name: regcred
```


