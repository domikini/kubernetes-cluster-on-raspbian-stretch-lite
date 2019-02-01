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

## Install Weave Net Driver

This installs Weave Net driver which is used for the cluster

`$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`
