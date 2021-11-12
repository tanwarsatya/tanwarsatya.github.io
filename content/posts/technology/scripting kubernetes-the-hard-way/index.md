---
title: "Scripting kubernetes-the-hard-way"
date: 2021-11-09T06:15:36-04:00
draft: false
socialshare: true
thumbnail: "img/thumbnails/kubernetes.png"
categories:
  - "kubernetes"
  - "technology"
  - "November, 2021"
  - "2021"
tags:
  - "kubernetes"
  - "automation"
  - "bash"
  - "CKA"
  - "CKS"
  - "CKAD"
---

During preparation for my **Certified Kubernetes Administrator** (CKA), **Certified Kubernetes Developer** (CKAD) and **Certified Kubernetes Security Specialist** (CKS) exams I realized the value of having a local Kubernetes cluster that can be used to experiment. Initially, I followed the instructions from [kubernetes-the-hard-way](https://github.com/kelseyhightower/kubernetes-the-hard-way) by **Kelsey Hightower** and manually built a local cluster on my Windows 10 laptop. It was a great starting point to experiment, but I faced many challenges with the manual installation. While experimenting much of my time was getting wasted in either identifying the changes that made the cluster unusable or in rebuilding the cluster again when not able to find the root cause of failure. This made me realize that I need an automated and repeatable solution to build a local cluster. As the process may allow me to focus on experimenting with various Kubernetes concepts instead of fixing or building clusters if some issue occurs.
<!--more-->

## The Plan

When I started to look around, I found many tools to build a local cluster, such as [k3sup](https://github.com/alexellis/k3sup), [k8s-tew](https://github.com/darxkies/k8s-tew) and [kOps](https://github.com/kubernetes/kops#installing), etc. All of them are straightforward tools, but I wanted a solution that will allow me to follow the steps defined as part of [kubernetes-the-hard-way](https://github.com/kelseyhightower/kubernetes-the-hard-way). I narrowed it down to using bash scripts to automate the required steps for the cluster installation. Scripting would save a lot of effort in creating a cluster or bringing back a non-functioning cluster to its starting state. In addition to that, with the scripts, it's easier to tinker and test various combinations for cluster creation with minimum effort.

---

## The Solution

The solution I developed includes multiple scripts for the cluster creation workflow. Having multiple scripts targeting specific steps allowed easy experimentation and fixing any specific issues with that step. The scripts are divided into different groups targeting the generation of cert authority and other required certificates, installing and configuring control plane components, installing and configuring worker node components, configuring network components, and validating the cluster installation by deploying. The scripts need to be executed in a specific order from a Linux shell.

> The scripts are available as [k8s-bare-metal](https://github.com/tanwarsatya/k8s-bare-metal) repository on **GitHub.com**

 Feel free to download them and experiment in your way. In subsequent sections, I will be covering different prerequisites required, the structure of scripts, and the functionality they cover.

### Prerequisites

Following are the prerequisites required for installation

>#### 1. Linux Shell to run bash scripts
>
>If using a Windows 10/11 based laptop or desktop you can [enable :desktop_computer: WSL2](https://docs.microsoft.com/en-us/windows/wsl/install#manual-installation-steps) to get a Linux shell.
>
>#### 2. Install/Update GIT on WSL
>
> Git comes installed on WSL still make sure it's installed and update it to the latest version by following this tutorial [Using GIT on WSL ](https://docs.microsoft.com/en-us/windows/wsl/tutorials/wsl-git)
>
>#### 3. Physical or Virtual machines running Ubuntu (18.04 or 20.04) with key based authentication
>
>If you don't have access to physical machines, VMs can be another alternative to meet this requirement. For installing VMs you can utilize [**Multipass**](https://multipass.run/) from Canonical.
>
>`:pushpin: Multipass is a mini-cloud on your workstation using native hypervisors of all the supported plaforms (Windows, macOS and Linux), it will give you an Ubuntu command line in just a click (“Open shell”) or a simple multipass shell command, or even a keyboard shortcut.'`
>
>Follow the steps given below to instantiate required VMs for the cluster:
>
>- [**Enable Hyper-V**](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/quick-start/enable-hyper-v) if you are using Windows based PC.
>
>- Install [**Multipass**](https://multipass.run/) on your PC.
>
>- Create an [external switch in Hyper-V](https://docs.microsoft.com/en-us/windows-server/virtualization/hyper-v/get-started/create-a-virtual-switch-for-hyper-v-virtual-machines) with a name - ***Multipass***. This will help in adding a secondary interface to the provisioned VMs with a local IP address and make VMs visible on the local network. We need to connect to VMs from the Linux shell to execute installation steps remotely.
>
>   ![external switch](img/external-switch.jpg "external switch")  
>
>
>- Generate OpenSSH private/public key pair to be used for key-based authentication for VMs. You can use [**ssh-keygen**](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent) command on WSL to generate private/public key pair. For simplicity use an empty passphrase.
>
>   ![sshkeygen](img/ssh-keygen.jpg "ssh keygen")  
>
>
>- Create a [**cloud_init.yaml**](https://cloudinit.readthedocs.io/en/latest/) file to be used for vm creation. Multipass allow using ***cloud_init.yaml*** to configure instantiated VMs. This way we can easily automate the required configuration of newly generated VMs. Follow the basic template example given below and replace the password and ssh_authorized_keys field values.
>
>
>     ```yaml
>     users:
>       - default
>         - name: k8suser
>           lock_passwd: false
>           plain_text_passwd: 'yourpassword'
>           sudo:  ALL=(ALL) NOPASSWD:ALL
>           ssh_authorized_keys:
>             - ##.pub key filecontent##
>     ssh_pwauth: True 
>     ```
>
>- Provision virtual machines using Multipass with the following commands. Feel free to change the parameters as required.
>
>     ```bash
>       multipass launch -n=k8s-master-1 --network name=multipass,mode=auto 
>       -c=2 -m=4G  -d=20G --cloud-init=cloud-init.yaml
>     ```
>
>     ```bash
>       multipass launch -n=k8s-node-1 --network name=multipass,mode=auto 
>       -c=2 -m=4G  -d=20G --cloud-init=cloud-init.yaml
>     ```
>
>     ```bash
>       multipass launch -n=k8s-node-2 --network name=multipass,mode=auto 
>       -c=2 -m=4G  -d=20G --cloud-init=cloud-init.yaml
>      ```
>
>- Make sure virtual machines are in started state and have a local IP address assigned to it. You should be able to ping it.
>
>    ![IP address](img/vm-ip-address.jpg "ip address")
>
> - Make sure you are able to log in via ssh with the private key from the key-pair generated in the above step.
>
>    ![vm ssh](img/vm-ssh.jpg "vm ssh")

### Structure

The scripts are arranged as per the below structure and grouped mainly under ***3 folders*** and various ***numbered scripts***. In the following section, I am going to review the purpose of the specific folder/files. You should be able to click on the link, and it will take you to the correct artifact in the repository.

```bash
k8s-bare-metal
|
└───cert-authority
│   └───bin
|   └───config
|   └───certs (generated during execution)
│   
└───control-plane
|   └───config
|   └───output (generated during execution)
|
└───worker-plane
|   └───config
|   └───output (generated during execution)
|
|
|   1_install_control_plane.sh
|   2_install_worker_plane.sh
|   3_install_network_plane.sh
|   4_verify_cluster.sh
|   variables.sh
```

- #### [cert-authority](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/cert-authority)

   This folder contains scripts and configuration files to generate required CA Root and certs used by k8s.

    ```bash
    k8s-bare-metal
    │
    └───cert-authority
    │   │   generate_ca_cert.sh
    │   │   README.md
    |   |
    │   └───config
    │       │   ca-config.json
    │       │   ca-csr.json    
    ```

    ***[config](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/cert-authority/config)*** folder inside it contain ca-config.json and ca-csr.json files to be used by cfssl/cfssljson to generate certs. Only CA Root will be generated by these configs.

   ***[generate_ca_cert.sh](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/cert-authority/generate_ca_cert.sh)*** script create a certs directory and uses config files and binaries from bin folder to generate CA inside the certs directory.

- #### [control-plane](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/control-plane)
  
  This folder contains scripts and configs to generate required YAML configuration and scripts to install control-plane components on Linux nodes

  ```bash
    k8s-bare-metal
    │
    └───control-plane
    |   │   generate_control_plane_certs.sh
    |   |   generate_control_plane_configs.sh
    |   |   generate_control_plane_services.sh
    |   |   install_etcd.sh
    |   |   install_haproxy.sh
    |   |   install_k8s.sh
    |   │   README.md   
    |   │
    │   └───config
    │       │   admin-csr.json
    │       │   encryption-config.json
    |       |   etcd-csr.json
    │       │   kube-apiserver-csr.json
    │       │   kube-controller-manager-csr.json
    │       │   kube-scheduler-csr.json
    │       │   kubelet-auth-role-binding.yaml
    │       │   kube-auth-role.yaml
    │       │   kubelet-auto-approve-csr-role-binding.yaml
    │       │   kubelet-auto-approve-renewals-role-binding.yaml
    │       │   kubelet-csr-role-binding.yaml
    │       │   kubernetes.default.svc.cluster.local
    │       │   service-account-csr.json 
    ```

  ***[config](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/control-plane/config)*** folder include various config and binding files required for control-plane components as shown above.

  ***[generate_control_plane_certs.sh](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/control-plane/generate_control_plane_certs.sh)*** bash file generates required certs for control-plane components and save the generated files inside ***output*** folder.

  ***[generate_control_plane_configs.sh](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/control-plane/generate_control_plane_configs.sh)*** bash file generates various config files with appropriate token and values based on parameters defined in variables.sh. The script generates following files

  - kube-controller-manager.kubeconfig
  - kube-scheduler.kubeconfig
  - kube-scheduler.yaml
  - admin.kubeconfig
  - haproxy.config (if more than one node is used for control-plane installation)
  - .kubeconfig

  ***[generate_control_plane_services.sh](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/control-plane/generate_control_plane_services.sh)*** bash file generates various service configuration files with appropriate token and values based on parameters defined in variables.sh. The script generates following files

  - etcd.service
  - kube-apiserver.service
  - kube-controller-manager.service
  - kube-scheduler.service

   ***[install_etcd.sh](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/control-plane/install_etcd.sh)*** bash file installs etcd components for control-plane nodes as per the nodes defined in variables file at parent folder level. This script uses various output artifacts generated by other scripts.

  ***[install_haproxy.sh](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/control-plane/install_haproxy.sh)*** bash file installs hxproxy components to access kube-apiserver when more than one control-plane nodes is specified in variables file

  ***[install_k8s.sh](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/control-plane/install_k8s.sh)*** bash file installs all the required components for control-plane on each control-plane nodes as specified in variables.sh

- #### [worker-plane](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/worker-plane)
  
  This folder contains scripts and configs to generate required YAML configuration and scripts to install worker-plane components on Linux nodes

  ```bash
  k8s-bare-metal
  |
  └───worker-plane
      |   generate_worker_plane_certs.sh
      |   generate_worker_plane_configs.sh
      |   generate_worker_plane_services.sh
      |   install_k8s.sh
      │   README.md
      │
      └───config
          │   config.toml
          │   kube-proxy-csr.json

  ```

  ***[config](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/worker-plane/config)*** folder include various config files required for worker-plane components as shown above.

  ***[generate_worker_plane_certs.sh](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/worker-plane/generate_worker_plane_certs.sh)***  bash file generates required certs for worker-plane components and save the generated files inside ***output*** folder.

  ***[generate_worker_plane_configs.sh](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/worker-plane/generate_worker_plane_configs.sh)*** bash file generates various config files with approproate token and values based on paramaters defined in variables.sh file at parent directory level. The script generates following files

  - kube-proxy.kubeconfig
  - kube-proxy-config.yaml
  - kubelet-config.yaml
  - kubelet.kubeconfig (for each worker node specified in variables.sh file)

  ***[generate_control_plane_services.sh](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/worker-plane/generate_control_plane_services.sh)*** bash file generates various service configuration files with appropriate token and values based on parameters defined in variables.sh. The script generates following files

  - kube-proxy.service
  - kubelet.service
  - containserd.service

   ***[install_k8s.sh](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/worker-plane/install_k8s.sh)*** bash file installs all the required components for worker-plane on each node as specified in variables.sh file

- #### [1_install_control_plane.sh](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/1_install_control_plane.sh)
  
  This script act as a workflow and uses various scripts from the control-plane folder to installs various control plane components on master nodes, it also generates CA root certs as a first item using the scripts from cert-authority.

- #### [2_install_worker_plane.sh](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/2_install_worker_plane.sh)
  
  This script act as a workflow and uses various scripts from the worker-plane folder to installs various worker plane components on worker nodes.

- #### [3_install_network_plane.sh](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/3_install_network_plane.sh)
  
  This script act as a workflow to installs networking cni plugin components on cluster. This script also configure the local kubeconfig to use kubectl from the local shell.

- #### [4_verify_cluster.sh](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/4_verify_cluster.sh)
  
  This script validates the cluster by checking various kubernetes services status, nodes status and also deploy random number of nginx pods to make sure cluster is up and running.

### Requirements

>Installation can be done with any of the following suggested configurations, and the Linux nodes' requirements directly depend on the configuration selected. Use any of the suggested node configurations for installation. If choosing a configuration that uses more nodes you can lower down the CORE, RAM, and DISK requirements mentioned below.
>
>`:computer: Node Configuration - 2 CORE, 4 GB RAM , 20 GB DISK`
>
>- ***Bare Minimum*** - 1 Linux node is required for the bare minimum installation of the Kubernetes cluster. This single node will host all the control-plane components and worker-plane components. Any applications/pod deployed will run on this node.
>
>- ***Suggested Minimal*** - 2 Linux nodes are required for the suggested minimal installation of the Kubernetes cluster. 1st Linux node will host all the control-plane and the 2nd Linux node will host all the worker-plane components. Based on configuration application/pods can be deployed on worker-node or both the nodes.
>
>- ***Standard*** - 3 Linux nodes are required for the standard installation of the Kubernetes cluster. 1st Linux node will host all the control-plane and the 2 Linux nodes will host the worker-plane components. 2 worker-nodes will be used for application/pods deployments.
>
>- ***Optimal*** - 6 Linux nodes are required for the optimal installation of the Kubernetes cluster. 2 Linux nodes will host control-plane components in a highly available configuration. 1 Linux will host haproxy for being a proxy to access kube-apiserver in a load balanced way. 3 worker nodes will be used for application/pods deployments.
>
>- ***Realistic*** - 8 Linux nodes are required for the realistic installation of the Kubernetes cluster. 2 Linux nodes will host ETCD components in a highly available configuration. 2 Linux nodes will host control-plane components in a highly available configuration. 1 Linux will host haproxy for being a proxy to access kube-apiserver in a load-balanced way. 3 worker nodes will be used for application/pods deployments.

## The Installation

For a successful installation follow the steps in the specified order

- Identify the configuration from the [Requirements](#requirements) section you like to use for your installation and based on that create several VMs using ***Multipass*** as mentioned in the [Prerequisite](#prerequisites) section. Skip creating VMs if you are using physical machines on your local network.

- Make sure you can ssh into the VMs created via key-based authentication, as the same method will be used by the scripts to remotely install and configure the components.

- Clone the [k8s-bare-metal](https://github.com/tanwarsatya/k8s-bare-metal) repository to your WSL environment.

  ![Clone repo](img/clone-repo.gif "clone repo")

- Copy the private key file you used to ssh into VMs, this key will be used by the scripts to remote into VMs specified. For this example, I have named the private key file as ***k8suser***.

  ![copy key](img/copy-key.gif "copy key")

- Review the [variables.sh](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/variables.sh) file at root directory level where you cloned the GitHub repository. And update the variables as per your configuration. The below example is for ***Standard Configuration*** using 3 Linux nodes to install the cluster.  For variable SSH_CERT specify the private key file name and make sure the private key is available at the same level of variables.sh file.

  ```bash
  # User name and key file for remote login on VMs for deploying cluster component
  SSH_USER="k8suser"
  SSH_CERT="k8suser"

  # Cluster Name
  CLUSTER_NAME="easy-k8s-cluster"

  #VM and Node Name
  #############################################
  declare -a CONTROL_PLANE_NODES=("k8s-master-1")
  declare -a CONTROL_PLANE_ETCD_NODES=("k8s-master-1")
  declare -a WORKER_PLANE_NODES=("k8s-node-1" "k8s-node-2")
  CLUSTER_API_LOAD_BALANCER="k8s-master-1"

  ```

- Execute the script ***1_install_control_plane.sh*** to install control-plane components

  ![install control plane](img/install-control-plane.gif "install ontrol plane")

- Execute the script ***2_install_worker_plane.sh*** to install control-plane components

  ![install worker plane](img/install-worker-plane.gif "install worker plane")

- Execute the script ***3_install_network_plane.sh*** to install control-plane components

  ![install network plane](img/install-network-plane.gif "install network plane")

- Execute the script ***4_verify_cluster.sh*** to verify cluster is running as expected

  ![verify cluster](img/verify-cluster.gif "verify cluster")

:boom: Awesome you should have a working cluster now. Sometimes the nodes may have a status not ready when you use lower configuration for your Linux nodes. Give it some time and use the kubectl to get the node status in 2-3 minutes. The script updates the local kubeconfig so that you can use kubectl from the WSL shell. Once you have a working cluster feel free to experiment and tinker with ***[variables.sh](https://github.com/tanwarsatya/k8s-bare-metal/blob/main/variables.sh)*** file and other scripts in different folders to get a better understanding of the steps required to install Kubernetes manually. Scripts are idempotent and can be executed multiple times, which will reset the cluster to its initial state. In case for any reason if the cluster is not showing ready status, please try to run the scripts again as that fixes an intermittent issue with the installation. You can run individual scripts as well to just install the specific component and verify them at your own pace. Fork the repo and change the scripts to learn more.

## Summary

[kubernetes-the hard-way](https://github.com/kelseyhightower/kubernetes-the-hard-way) by **Kelsey Hightower** is the gold standard for learning internals of Kubernetes working. You can avoid the time-consuming nature of Kubernetes installation by utilizing the scripts I developed. These scripts closely follow the **Kelsey Hightower's** steps and allow any kind of experimentation you want to try. Scripts are available as part of [k8s-bare-metal](https://github.com/tanwarsatya/k8s-bare-metal) repository on GitHub.com. I hope this will save some of your valuable time and don't forget to share this blog and scripts if this helped you in your learning :thumbsup:.
