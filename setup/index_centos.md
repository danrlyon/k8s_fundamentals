# Install Kubernetes on AWS
## Log into all servers 
### MacOS 
Run the following commands in a terminal 
```
chmod 600 /path/to/lab.pem
ssh -i /path/to/lab.pem ubuntu@<server IP>
```

### Windows 
Open Putty and configure a new session. 
  
![](index/C4EC1E64-175D-4C84-8C49-D938337FA35A%208.png)


Expand “Connection/SSH/Auth and then specify the PPK file 

![](index/6FFB137C-1AD8-48A1-97E6-F5F6DA4BC55B%208.png)

 Now save your session 

![](index/FD3BA694-FD69-4C86-8EAF-4D5FC813EABA%208.png)

## Create Instances
Use the CentOS 7 (x86_64) - with Updates HVM AMI for your region
T2.medium for size
Use the default security group for all instances - make sure all ports are open for in/out traffic
Both Instances should be part of the same subnet (in the same VPC)
Add the EFS file system if created in class

Create 2 instances: 1 master/1 worker - set their name label to your initials-master/your userid-worker1 (e.g jkidd-master/jkidd-worker1)

## Install Kubernetes on all servers
When opening an SSH connection use the centos user.
Following commands must be run as the root user. To become root run: 
```
sudo su - 
```
### Install Docker Runtime 
NOTE: This will have to be done on all servers
```
yum install -y yum-utils
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce docker-ce-cli containerd.io
systemctl start docker
systemctl enable docker
```
So that we can run Docker without being priviledged:
```
groupadd docker
usermod -aG docker $USER
```
You will need to logout and back in....

To prove everything is working:
```
docker run hello-world
```

### Install packages required for Kubernetes on all servers as the root user
Letting iptables see bridged traffic
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

Make sure the security group being used by the master and all worker nodes has all port open.

Create Kubernetes repository by running the following as one command.
```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```
### Set SELinux in permissive mode (effectively disabling it)
```
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

Now that you've added the repository install the packages
```
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable --now kubelet
```
***Note: kubectl does not need to be installed on worker nodes

The kubelet is now restarting every few seconds, as it waits in a `crashloop` for `kubeadm` to tell it what to do.

### Initialize the Master 
Run the following command on the master node to initialize 
```
kubeadm init --kubernetes-version=1.17.0 --ignore-preflight-errors=all
```

If everything was successful output will contain 
````
Your Kubernetes master has initialized successfully!
````

Note the `kubeadm join...` command, it will be needed later on. Save this in a text editor.

Exit to ubuntu user 
```
exit
```

Now configure server so you can interact with Kubernetes as the unprivileged user. 
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Run following on the master to enable IP forwarding to IPTables.
```
sudo sysctl net.bridge.bridge-nf-call-iptables=1
```

### Pod overlay network
Install a Pod network on the master node
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

Wait until `coredns` pod is in a `running` state
```
kubectl get pods -n kube-system
```

### Join nodes to cluster 
Log into each of the worker nodes and run the join command from `kubeadm init` master output. 
```
sudo kubeadm join <command from kubeadm init output> --ignore-preflight-errors=all
```

To confirm nodes have joined successfully log back into master and run 
```
watch kubectl get nodes 
````

When they are in a `Ready` state the cluster is online and nodes have been joined. 

# Congrats! 
