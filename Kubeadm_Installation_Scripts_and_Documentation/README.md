# Kubeadm Installation Guide

This guide outlines the steps needed to set up a Kubernetes cluster using `kubeadm`.

## Prerequisites

- Ubuntu OS (Xenial or later)
- `sudo` privileges
- Internet access
- t2.medium instance type or higher

---

## AWS Setup

1. Ensure that all instances are in the same **Security Group**.
2. Expose port **6443** in the **Security Group** to allow worker nodes to join the cluster.
3. Expose port **22** in the **Security Group** to allows SSH access to manage the instance..


## To do above setup, follow below provided steps

### Step 1: Identify or Create a Security Group

1. **Log in to the AWS Management Console**:
    - Go to the **EC2 Dashboard**.

2. **Locate Security Groups**:
    - In the left menu under **Network & Security**, click on **Security Groups**.

3. **Create a New Security Group**:
    - Click on **Create Security Group**.
    - Provide the following details:
      - **Name**: (e.g., `Kubernetes-Cluster-SG`)
      - **Description**: A brief description for the security group (mandatory)
      - **VPC**: Select the appropriate VPC for your instances (default is acceptable)

4. **Add Rules to the Security Group**:
    - **Allow SSH Traffic (Port 22)**:
      - **Type**: SSH
      - **Port Range**: `22`
      - **Source**: `0.0.0.0/0` (Anywhere) or your specific IP
    
    - **Allow Kubernetes API Traffic (Port 6443)**:
      - **Type**: Custom TCP
      - **Port Range**: `6443`
      - **Source**: `0.0.0.0/0` (Anywhere) or specific IP ranges

5. **Save the Rules**:
    - Click on **Create Security Group** to save the settings.

### Step 2: Select the Security Group While Creating Instances

- When launching EC2 instances:
  - Under **Configure Security Group**, select the existing security group (`Kubernetes-Cluster-SG`)

> Note: Security group settings can be updated later as needed.

---


## Execute on Both "Master" & "Worker" Nodes

1. **Disable Swap**: Required for Kubernetes to function correctly.
    ```bash
    sudo swapoff -a
    sudo sed -i '/ swap / s/^/#/' /etc/fstab
  ```

2. **Load Necessary Kernel Module and sysctl settings**: Required for Kubernetes networking.
    ```bash
    # Load the br_netfilter module on boot
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    br_netfilter
    EOF

    # Apply required sysctl params for Kubernetes networking
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-ip6tables = 1
    net.bridge.bridge-nf-call-iptables = 1
    net.ipv4.ip_forward = 1
    EOF

    # Load the module immediately
    sudo modprobe br_netfilter

    # Apply sysctl params without reboot
    sudo sysctl --system
 ```

3. ** Install Docker**: 
    ```bash
    sudo apt-get update
    sudo apt-get install -y ca-certificates curl gnupg lsb-release gnupg2 software-properties-common apt-transport-https

    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

    sudo add-apt-repository \
    "deb [arch=$(dpkg --print-architecture)] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable"

    sudo apt-get update
    sudo apt-get install -y docker-ce docker-ce-cli containerd.io
```

4. **Install cri-dockerd**:
    ```bash
    sudo apt-get install -y git golang-go

    git clone https://github.com/Mirantis/cri-dockerd.git
    cd cri-dockerd
    mkdir bin
    go build -o bin/cri-dockerd

    sudo mv bin/cri-dockerd /usr/local/bin/

    sudo cp -a packaging/systemd/* /etc/systemd/system
    sudo sed -i 's:/usr/bin/cri-dockerd:/usr/local/bin/cri-dockerd:' /etc/systemd/system/cri-docker.service

    sudo systemctl daemon-reexec
    sudo systemctl daemon-reload
    sudo systemctl enable cri-docker.service
    sudo systemctl enable --now cri-docker.socket

    ```

5. **Install Kubernetes Components**:
    ```bash
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl gpg

    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

    sudo apt-get update
    sudo apt-get install -y kubelet kubeadm kubectl
    sudo apt-mark hold kubelet kubeadm kubectl
    ```

## Execute ONLY on the "Master" Node

1. **Initialize the Cluster**:
    ```bash
    sudo kubeadm init --cri-socket unix:///var/run/cri-dockerd.sock --pod-network-cidr=10.244.0.0/16

    ```

2. **Set Up Local kubeconfig**:
    ```bash
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

    ```

3. **Install a Network Plugin (Calico)**:
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

    ```

4. **Generate Join Command**:
    ```bash
    sudo kubeadm join <MASTER-IP>:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --cri-socket unix:///var/run/cri-dockerd.sock

    ```

> Copy this generated token for next command.

---

## Execute on ALL of your Worker Nodes

1. Perform pre-flight checks:
    ```bash
    sudo kubeadm reset --cri-socket unix:///var/run/cri-dockerd.sock

    ```

2. Paste the join command you got from the master node and append `--v=5` at the end:
    ```bash
    sudo kubeadm join 172.31.16.114:6443 \
  --token qou6pp.fjzo6ayko9mydj2c \
  --discovery-token-ca-cert-hash sha256:febcf51565ee94ca1e97c04d61810100b233ca65f9399ada5265fcff0595216b \
  --cri-socket unix:///var/run/cri-dockerd.sock --v=5

    ```

    > **Note**: When pasting the join command from the master node:
    > 1. Add `sudo` at the beginning of the command
    > 2. Add `--v=5` at the end
    >
    > Example format:
    > ```bash
    > sudo <paste-join-command-here> --v=5
    > ```

---

## Verify Cluster Connection

**On Master Node:**

```bash
kubectl get nodes

```

   <img src="https://raw.githubusercontent.com/faizan35/kubernetes_cluster_with_kubeadm/main/Img/nodes-connected.png" width="70%">

---

## Verify Container Status on Worker Node
<img src="https://github.com/user-attachments/assets/c3d3732f-5c99-4a27-a574-86bc7ae5a933" width="70%">


