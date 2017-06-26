# Setting up Kubernetes on CentOS 7

### 1) Introduction

This document describes all the steps that are required to install and set up Kubernetes cluster on CentOS 7 machines. This set up tutorial is divided into three parts: first part discusses the lab machines set up that we follow, second part showcases installation steps required to set up Kubernetes cluster on CentOS 7 and finally third part discusses all the problems that may occur during the set up and potential fixes for them.

### 2) Environment Setup
Our set up consists of 3 bare metal machines which have CentOS 7 installed in them. You may also want to set up this cluster using virtual machines. We make sure that these three machines are on same subnet and they can ping each other over the network.    

	So, below are our nodes including master node and worker nodes:     
  10.23.114.120 node1     -- Master Node   
  10.23.114.194 node2 	  -- Worker Node 1    
  10.23.114.118 node3	    -- Worker Node 2     
  
Before you proceed to installation and setup steps next, please make sure you have made changes to `/etc/hosts` to include all the nodes by hostnames and IP. This will make sure we will have connectivity among machines by hostnames.

### 3) Installation and Setup    
Kubernetes have their official guide for setting up Kubernetes cluster on CentOS 7 machines and it is located [here](http://https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/). To execute steps below, you may want to acquire root user privileges. Now, if you are sitting behind a proxy, then you may want to add `http_proxy` variable in your system. You may also include the proxy variable by editing `/etc/yum.conf` file and adding “`proxy=<http://proxy.something.com:port>`” at the end of the file.  

* **Installation of Docker**   
On each of your three machines- including master node and two worker nodes- install Docker. Please note that Kubernetes’ Kubeadm tool is not extensively tested on Docker versions 1.13 and 17.03+. Therefore, to avoid future problems with our set up we specifically install Docker version 1.12.    

  * Add Docker Repo   

  ```bash
  $ sudo yum install -y yum-utils    
  $ yum-config-manager \
    --add-repo \
    https://docs.docker.com/v1.13/engine/installation/linux/repo_files/centos/docker.repo
  $ yum makecache fast   
  ```   
  
  There is a good chance that you might not be able to connect to docs.docker.com, so you may want to install docker.repo on your local   windows machine first and then transfer that repo file to /etc/yum.repos.d/ directory in each of three machines using tools like      WinSCP (on Windows) or create a new repo file.   
  
  * List all packages   
  Now, we check all versions of Docker that are offered in the repo we added above. As we know that Kubernetes kubeadm is not tested extensively on Docker 1.13 and 17.03+ versions, we install Docker 1.12 version.    
  ```bash
  
  $ yum list docker-engine.x86_64  --showduplicates |sort –r

  docker-engine.x86_64     1.13.1-1.el7.centos              docker-main   
  docker-engine.x86_64     1.12.6-1.el7.centos              docker-main   
  docker-engine.x86_64     1.11.2-1.el7.centos              docker-main   
  ```
  We will install Docker version 1.12 on all our machines in the setup.   

  * Install and start Docker on all machines   
  ```bash
  $ yum install -y docker-engine-1.12.6   
  $ systemctl start docker
  $ systemctl enable docker   
  ```
  
    
* **Install and Set Up Kubernetes Components**   
Next, we will install core components of Kubernetes that are essential for our Kubernetes cluster to work. These components include kubectl, kubelet and kubeadm. We will also install other components that will be important for our set up.   
  * Add Kubernetes Repo   
  ```bash
  $ cat <<EOF > /etc/yum.repos.d/kubernetes.repo

  [kubernetes]
  name=Kubernetes
  baseurl=http://yum.kubernetes.io/repos/kubernetes-el7-x86_64
  enabled=1
  gpgcheck=1
  repo_gpgcheck=1
  gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
  EOF
  ```   
  Now, we check the version of packages offered by this repo.   
  
  * List Packages   
  ```bash
  $ yum list kubeadm  --showduplicates |sort -r

  kubeadm.x86_64            1.6.5-0                        kubernetes
  kubeadm.x86_64            1.6.4-0                        kubernetes

  $ yum list kubelet  --showduplicates |sort -r

  kubelet.x86_64            1.6.5-0                        kubernetes
  kubelet.x86_64            1.6.4-0                        kubernetes
  kubelet.x86_64            1.6.3-0                        kubernetes

  $ yum list kubectl  --showduplicates |sort -r

  kubectl.x86_64            1.6.5-0                        kubernetes
  kubectl.x86_64            1.6.4-0                        kubernetes
  kubectl.x86_64            1.6.3-0                        kubernetes


  $ yum list kubernetes-cni  --showduplicates |sort -r

  kubernetes-cni.x86_64     0.5.1-0                        kubernetes
  ```   
  Next on, we install latest versions of these packages in following step.   
  
  * Install `kubeadm`, `kubectl` and CNI Plugin
  ```bash
  $ setenforce 0

  $ yum install -y kubelet kubeadm kubectl kubernetes-cni
  ...
  ... 

  Installed:
	       kubeadm.x86_64 0:1.6.5-0              
         kubectl.x86_64 0:1.6.5-0
         kubelet.x86_64 0:1.6.5-0
         kubernetes-cni.x86_64 0:0.5.1-0

  Complete!

  $ systemctl enable kubelet.service
  ```   
  
  * Initialize Kubernetes Cluster   
  ```bash
  kubeadm init --kubernetes-version=v1.6.5 --pod-network cidr=10.245.0.0/16 --apiserver-advertise-address=10.23.114.120
  ```    
  This initializes the cluster and our master node listens on address 10.23.114.120.    
  
  * Start Using the Cluster    
  To start using your cluster, please follow below commands as a regular user.    
  ```bash
  $ sudo cp /etc/kubernetes/admin.conf $HOME/
  $ sudo chown $(id -u):$(id -g) $HOME/admin.conf
  $ export KUBECONFIG=$HOME/admin.conf
  ```    
  
  * Join Worker Nodes to Cluster    
  After you start the cluster with kubeadm init command and the init command runs successfully, you should see a command to join the cluster using a token mentioned. Run that command as root on all the worker nodes so that they join the cluster.    
   ```bash
  $ kubeadm join --token e7986d.e440de5882342711 10.23.114.120:6443
  ```
  Now, to check if all nodes have joined the cluster please run following command on master node.   
  
   ```bash
  $ kubectl get nodes

  NAME      STATUS     AGE       VERSION
  node1     Ready      3m        v1.6.5
  node2     Ready      2m        v1.6.5
  node3     Ready      52s       v1.6.5
  ```   
  
  Your Kubernetes cluster is good to go! Next step, we will deploy pods in our cluster.    
  
  * Deploy PODs

  




