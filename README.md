# User Manual: Connecting to a Kubernetes Cluster and Deploying Jupyter Notebook with PyTorch on NVIDIA DGX

Stockholm Medical Artificial Intelligence and Learning Environments (SMAILE) uses a **Kubernetes (K8S) cluster** hosted in the **Karolinska Institutet (KI) data centre**. This cluster is **not accessible through the internet**, meaning users must either:
- Be on the **KI network**, or
- Use the **KI VPN** if accessing remotely.

## Table of Contents

1. [Introduction to Kubernetes and Containers](#1-introduction-to-kubernetes-and-containers)
2. [Introduction to Docker and Dockerfiles](#2-introduction-to-docker-and-dockerfiles)
3. [Connecting to the SMAILE K8S Cluster](#3-connecting-to-the-smaile-k8s-cluster)
4. [Managing Kubernetes Credentials](#4-managing-kubernetes-credentials)
5. [Installation and Setup Guide](#5-installation-and-setup-guide)
6. [Deploying Jupyter Notebook with PyTorch Using kubectl](#6-deploying-jupyter-notebook-with-pytorch-using-kubectl)
7. [Managing Persistent Storage](#7-managing-persistent-storage)
8. [Essential Kubernetes Commands](#8-essential-kubernetes-commands)
9. [Troubleshooting](#9-troubleshooting)
10. [Kubernetes Ecosystem and Relationships](#10-kubernetes-ecosystem-and-relationships)
11. [Summary](#11-summary)

## 1. Introduction to Kubernetes and Containers

### What is Kubernetes?
Kubernetes (K8s) is an open-source container orchestration platform that automates deploying, managing, and scaling applications. It enables efficient use of resources by distributing workloads across clusters of machines.

### Key Kubernetes Concepts
- **Pods:** The smallest deployable units in Kubernetes that contain one or more containers
- **Deployments:** Controllers that manage pod lifecycles and enable rolling updates
- **Services:** Network endpoints that expose pods to the network
- **Nodes:** Worker machines (physical or virtual) that run pods
- **Namespaces:** Virtual clusters for resource isolation and organization

### Benefits of Kubernetes
- **Scalability:** Applications can be scaled up or down automatically.
- **Self-Healing:** Kubernetes restarts failed containers and replaces them when needed.
- **Resource Efficiency:** Ensures optimal use of CPU, memory, and GPUs.
- **Load Balancing:** Distributes network traffic across pods efficiently.
- **Declarative Configuration:** Applications are described in YAML manifests, making them reproducible.

### What are Containers?
Containers package applications and their dependencies into isolated environments, making them portable across different computing environments. Popular container runtime environments include Docker.

## 2. Introduction to Docker and Dockerfiles

### What is Docker?
Docker is a platform for developing, shipping, and running applications in containers. Containers ensure that applications run the same way regardless of where they are deployed.

### Container vs. Virtual Machine
Containers share the host OS kernel but have isolated processes and filesystems. This makes them more lightweight than virtual machines, which include a full OS instance.

### Using Default vs. Custom Docker Images
For deployments, you can either use a **default container image** such as `pytorch/pytorch:latest`, which includes the necessary dependencies, or **create a custom Docker image** tailored to specific development needs.

### Understanding Dockerfiles
A Dockerfile is a script containing a set of instructions to assemble a container image. It defines:
- **Base image:** The starting point (e.g., Ubuntu, NVIDIA PyTorch image).
- **Dependencies:** Software, libraries, and tools required.
- **Commands:** Instructions to run the application (e.g., Jupyter Notebook).

### Example Dockerfile for PyTorch with Jupyter
```Dockerfile
FROM pytorch/pytorch:latest

# Install additional packages
RUN pip install --no-cache-dir \
    matplotlib \
    pandas \
    scikit-learn \
    seaborn \
    jupyterlab

# Set working directory
WORKDIR /workspace

# Expose Jupyter port
EXPOSE 8888

# Start Jupyter when container launches
CMD ["jupyter", "lab", "--ip=0.0.0.0", "--port=8888", "--no-browser", "--allow-root", "--NotebookApp.token=''", "--NotebookApp.password=''"]
```

### Building and Pushing Docker Images
```sh
# Build the Docker image
docker build -t your-registry/pytorch-jupyter:v1 .

# Push the image to a registry
docker push your-registry/pytorch-jupyter:v1
```

## 3. Connecting to the SMAILE K8S Cluster
To connect to the **SMAILE Kubernetes Cluster**, you must either:
- Be on the **KI network** or
- Use the **KI VPN** if accessing remotely.

### Setting up VPN Access
For detailed instructions on setting up and using the KI VPN service, please refer to the official KI documentation:
[KI VPN Service Documentation](https://staff.ki.se/tools-and-support/it-and-telephony/tools-for-working-off-campus/vpn-service-ki-vpn)

The documentation covers:
- How to request VPN access
- Installation of the VPN client
- Connection instructions
- Troubleshooting common connection issues

## 4. Managing Kubernetes Credentials
Users receive Kubernetes credentials as a YAML file, typically named `kubeconfig.yaml`. To use it, place it in the appropriate directory:

### Linux/macOS
```sh
mkdir -p ~/.kube
mv kubeconfig.yaml ~/.kube/config
export KUBECONFIG=~/.kube/config
```

### Windows (PowerShell)
```powershell
mkdir -Force $HOME\.kube
Move-Item kubeconfig.yaml $HOME\.kube\config
$env:KUBECONFIG="$HOME\.kube\config"
```

To make the KUBECONFIG environment variable persistent on Windows, you can set it in the system environment variables:
1. Search for "Edit the system environment variables" in Windows search
2. Click "Environment Variables"
3. Under "User variables", click "New"
4. Variable name: `KUBECONFIG`
5. Variable value: `%USERPROFILE%\.kube\config`

To test that Kubernetes is correctly configured, run:

```sh
kubectl get pods
```

If the connection is successful, it will list the running pods in the cluster.

## 5. Installation and Setup Guide

### Step 1: Install Chocolatey (Windows only)
Chocolatey is a package manager for Windows, useful for installing software such as kubectl.

To install Chocolatey, open PowerShell as Administrator and run:

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

### Step 2: Install kubectl
Kubectl is the command-line tool for interacting with Kubernetes clusters.

#### Windows:
Using Chocolatey (recommended):
```powershell
choco install kubernetes-cli
```

Alternatively, follow the official installation guide: [Install kubectl on Windows](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)

#### macOS:
```sh
brew install kubectl
```

#### Ubuntu:
```sh
sudo apt update && sudo apt install -y kubectl
```

After installation, verify it is working by running:

```sh
kubectl version --client
```

### Step 3: Copy Kubernetes Credentials
Once `kubectl` is installed, copy your `kubeconfig.yaml` file to the appropriate directory:

```sh
mkdir -p ~/.kube
mv kubeconfig.yaml ~/.kube/config
export KUBECONFIG=~/.kube/config
```

For Windows:

```powershell
mkdir -Force $HOME\.kube
Move-Item kubeconfig.yaml $HOME\.kube\config
$env:KUBECONFIG="$HOME\.kube\config"
```

To verify that credentials work, run:

```sh
kubectl get pods
```

If the connection is successful, it will list the running pods.

## 6. Deploying Jupyter Notebook with PyTorch Using kubectl

### Step 1: Create Kubernetes YAML Files
Create a file named `jupyter-deployment.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pytorch-jupyter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pytorch-jupyter
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: pytorch-jupyter
    spec:
      enableServiceLinks: false
      containers:
      - name: pytorch-jupyter
        image: pytorch/pytorch:latest
        command:
          - /bin/bash
          - -c
          - |
            pip install jupyter notebook
            jupyter notebook --ip=0.0.0.0 --port=8888 --allow-root
        ports:
        - containerPort: 8888
        resources:
          limits:
            nvidia.com/gpu: 1
            memory: "16Gi"
            cpu: "4"
          requests:
            memory: "4Gi"
            cpu: "2"
        volumeMounts:
        - mountPath: /workspace
          name: data
        - mountPath: /dev/shm
          name: shm
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: jupyter-data-pvc
      - name: shm
        emptyDir:
          medium: Memory
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jupyter-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
---
apiVersion: v1
kind: Service
metadata:
  name: pytorch-jupyter-service
spec:
  selector:
    app: pytorch-jupyter
  ports:
  - port: 8888
    targetPort: 8888
  type: ClusterIP
```

### Step 2: Apply the YAML Files
Apply the YAML configuration to create the Kubernetes resources:

```sh
kubectl apply -f jupyter-deployment.yaml
```

### Step 3: Verify Deployment
Check the status of your deployment:

```sh
kubectl get pods
```

Wait until the pod status shows `Running`.

### Step 4: Get Jupyter Notebook Access Token
To access Jupyter Notebook, you need to get the access token from the pod logs:

```sh
kubectl logs $(kubectl get pods -l app=pytorch-jupyter -o name)
```

Look for a line that shows the token URL, which should look something like:
```
http://127.0.0.1:8888/?token=abcdef123456...
```

Copy the token value (the part after `token=`).

### Step 5: Set Up Port Forwarding
Set up port forwarding to access Jupyter Notebook:

```sh
kubectl port-forward service/pytorch-jupyter-service 8888:8888
```

### Step 6: Access Jupyter Notebook
Open your browser and go to:
```
http://localhost:8888/?token=<your-token>
```

Replace `<your-token>` with the token you copied from the logs.

### Step 7: Managing Your Deployment

#### Scale Down (Release GPU Resources)
When you're not using the notebook, release resources by scaling down:

```sh
kubectl scale deployment pytorch-jupyter --replicas=0
```

#### Scale Up (Resume Work)
When you want to resume work:

```sh
kubectl scale deployment pytorch-jupyter --replicas=1
```

#### Delete Deployment
To completely remove the deployment:

```sh
kubectl delete -f jupyter-deployment.yaml
```

Note: This will not delete the Persistent Volume Claim if it's being used.



## 7. Managing Persistent Storage

### Understanding Persistent Volumes in Kubernetes
Persistent Volumes (PVs) and Persistent Volume Claims (PVCs) provide storage that persists beyond the lifecycle of pods. This is crucial for saving work in Jupyter Notebooks and MATLAB.

### Types of Storage Classes
A cluster might offer different storage classes, such as:
- **standard**: General-purpose storage
- **ssd**: SSD-backed storage for better performance
- **shared**: Storage accessible from multiple pods simultaneously

You can view available storage classes with:
```sh
kubectl get storageclass
```

### Checking Storage Status
To view your Persistent Volume Claims:
```sh
kubectl get pvc
```

To see detailed information about a specific PVC:
```sh
kubectl describe pvc <pvc-name>
```

### 7.1 Copying Files to a Persistent Volume Claim (PVC)
In Kubernetes, **Persistent Volume Claims (PVCs)** provide storage that persists beyond the lifecycle of pods. If you need to copy files from your local machine to a PVC-mounted directory inside a pod, you can use the `kubectl cp` command.

#### Using `kubectl cp`
The `kubectl cp` command allows you to copy files between your local machine and a running pod.

#### Step 1: Find the Pod Name
To locate the pod using your PVC, run:

```sh
kubectl get pods
```

If you need more details about the storage mounts in a specific pod:

```sh
kubectl describe pod <pod-name>
```

#### Step 2: Copy Files to the Pod
To copy a local file to a pod that has access to the PVC:

```sh
kubectl cp /path/to/local/file <pod-name>:/path/in/pod
```

For example, if your pod is named `pytorch-jupyter` and your PVC is mounted at `/workspace`, you can copy a local dataset like this:

```sh
kubectl cp dataset.csv pytorch-jupyter:/workspace/dataset.csv
```

To copy an entire directory:

```sh
kubectl cp /path/to/local/dir <pod-name>:/workspace/
```

#### Step 3: Verify the File
After copying, log into the pod to verify:

```sh
kubectl exec -it <pod-name> -- ls -l /workspace
```

#### Troubleshooting
* **Permission Issues:** Ensure the target directory in the pod allows write access.
* **File Overwrites:** Existing files may be overwritten without warning.
* **Slow Transfers:** Large files may take time due to Kubernetes network overhead.

This method enables easy file transfers when working with Kubernetes **Persistent Volumes**, improving workflows for data science and research environments.

## 8. Essential Kubernetes Commands

### Get and View Resources

```sh
# Get all resources in the current namespace
kubectl get all

# Get all resources including persistent volume claims
kubectl get all,pvc

# Get pods with more details
kubectl get pods -o wide

# Get all resources across all namespaces
kubectl get all --all-namespaces

# List resources with labels
kubectl get pods --show-labels

# Filter resources by label
kubectl get pods -l app=pytorch-jupyter

# Watch resources in real-time
kubectl get pods --watch
```

### Pod Management
```sh
# List all pods
kubectl get pods

# Get detailed information about a pod
kubectl describe pod <pod-name>

# View pod logs
kubectl logs <pod-name>

# Follow/stream pod logs in real-time
kubectl logs -f <pod-name>

# Execute a command in a pod
kubectl exec -it <pod-name> -- /bin/bash

# Copy files to/from a pod
kubectl cp <local-path> <pod-name>:<pod-path>
kubectl cp <pod-name>:<pod-path> <local-path>
```

### Deployment Management
```sh
# List all deployments
kubectl get deployments

# Scale a deployment
kubectl scale deployment <deployment-name> --replicas=<number>

# Edit a deployment
kubectl edit deployment <deployment-name>

# Restart a deployment (rolling restart)
kubectl rollout restart deployment <deployment-name>

# View deployment rollout status
kubectl rollout status deployment <deployment-name>

# Pause/resume a deployment rollout
kubectl rollout pause deployment <deployment-name>
kubectl rollout resume deployment <deployment-name>
```

### Delete Resources
```sh
# Delete a specific pod
kubectl delete pod <pod-name>

# Delete a deployment
kubectl delete deployment <deployment-name>

# Delete a service
kubectl delete service <service-name>

# Delete multiple types of resources
kubectl delete pod,service <pod-name>,<service-name>

# Delete resources using label selectors
kubectl delete pods,services -l app=pytorch-jupyter

# Delete all pods (will be recreated if managed by deployments)
kubectl delete pods --all

# Delete all resources in the current namespace
kubectl delete all --all

# Force delete a stuck resource
kubectl delete pod <pod-name> --grace-period=0 --force
```

### Service Management
```sh
# List all services
kubectl get services

# Get detailed information about a service
kubectl describe service <service-name>

# Access a service locally via port forwarding
kubectl port-forward service/<service-name> <local-port>:<service-port>

# Port forward to a specific pod
kubectl port-forward pod/<pod-name> <local-port>:<pod-port>
```

### Persistent Volume Management
```sh
# List all persistent volume claims
kubectl get pvc

# Get detailed information about a PVC
kubectl describe pvc <pvc-name>

# List all persistent volumes
kubectl get pv

# Delete a PVC (caution: this may delete data)
kubectl delete pvc <pvc-name>

# Get storage classes
kubectl get storageclass
```

### Working with YAML files
```sh
# Create resources from a YAML file
kubectl apply -f <filename.yaml>

# Delete resources defined in a YAML file
kubectl delete -f <filename.yaml>

# Validate a YAML file without applying it
kubectl apply -f <filename.yaml> --dry-run=client

# Export a resource as YAML
kubectl get deployment <deployment-name> -o yaml > exported.yaml

# View the manifest of a running resource
kubectl get deployment <deployment-name> -o yaml
```

### Namespace Management
```sh
# List all namespaces
kubectl get namespaces

# Create a new namespace
kubectl create namespace <namespace-name>

# Set default namespace for kubectl
kubectl config set-context --current --namespace=<namespace-name>

# View resources in a specific namespace
kubectl get pods -n <namespace-name>
```

### Cluster Information
```sh
# Display cluster information
kubectl cluster-info

# Check API resources available
kubectl api-resources

# Check component status
kubectl get componentstatuses

# Show nodes with details
kubectl get nodes -o wide

# List events in the cluster
kubectl get events
```

## 9. Troubleshooting

### Common Issues and Solutions

#### Issue: Cannot connect to the cluster
- **Check VPN connection** - Ensure you are connected to the KI VPN
- **Verify kubeconfig** - Make sure your kubeconfig file is correctly located at `~/.kube/config`
- **Check credentials** - Run `kubectl config view` to ensure your configuration is correct
- **Test connectivity** - Try `kubectl cluster-info` to verify cluster connection

#### Issue: Pod stays in "Pending" state
- **Check resource availability** - GPUs might be fully utilized
- **Inspect pod events** - Run `kubectl describe pod <pod-name>` to see the events
- **Verify resource requests** - Your pod might be requesting more resources than available
- **Check node status** - Run `kubectl get nodes` to see if any nodes are available

#### Issue: Cannot access Jupyter or MATLAB
- **Verify port forwarding** - Ensure port forwarding is active
- **Check service status** - Run `kubectl get service <service-name>`
- **Inspect pod logs** - Run `kubectl logs <pod-name>` to check for application errors
- **Check network connectivity** - Make sure no firewall is blocking the connection

#### Issue: Lost data after pod restart
- **Verify persistence configuration** - Ensure persistence is enabled in your YAML files
- **Check PVC status** - Run `kubectl get pvc` to verify the persistent volume claim exists
- **Inspect mount paths** - Verify the mount path in your container matches your configuration
- **Examine events** - Run `kubectl get events` to see if there were any storage-related issues

#### Issue: Cannot find Jupyter token
- **Check logs again** - Sometimes the token may not appear in the first few lines of logs
- **Use grep** - Try `kubectl logs <pod-name> | grep token` to find the token
- **Restart the pod** - Delete and recreate the pod to generate a new token
- **Configure token-less access** - Modify your YAML to use `--NotebookApp.token=''` in the Jupyter command

### Getting Help
For issues not covered in this troubleshooting guide, please contact your system administrator or refer to the official Kubernetes documentation at [kubernetes.io](https://kubernetes.io/docs/home/).

## 10. Kubernetes Ecosystem and Relationships

The following diagram illustrates the relationships between different components of the Kubernetes ecosystem and the tools used to interact with it:

![Kubernetes Ecosystem and Relationships](https://raw.githubusercontent.com/ki-smile/nvidia-charts/main/k8s-concepts-diagram.svg)

### Key Relationships Explained

- **Docker** builds container images used by pods
- **kubectl** is the command-line tool for interacting directly with the cluster
- **Deployments** manage multiple pod replicas and handle rolling updates
- **Services** expose pods to the network via stable endpoints
- **Persistent Volumes** provide storage that persists beyond pod lifecycles
- **YAML files** define the desired state of Kubernetes resources

## 11. Summary
By following this guide, you can efficiently deploy and manage Jupyter Notebook and MATLAB instances on the SMAILE Kubernetes cluster powered by NVIDIA GPUs.

Key points to remember:
- Connect to the KI VPN when accessing the cluster remotely
- Use `kubectl` and YAML files for deployment and management
- Get the Jupyter token from the pod logs to access the notebook
- Set up port forwarding to access services from your local machine
- Properly configure persistent storage to avoid data loss
- Scale down deployments when not in use to free up resources for other users
- Understand basic Kubernetes commands for managing your deployment

For more information, refer to the official documentation:
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/overview.html)
