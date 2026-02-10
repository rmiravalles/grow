# 🧪 Getting Started - Setting up my Kubernetes environment

Welcome! This is your first notebook entry in the GROW program.  
Use this space to document the setup of your **Kubernetes learning environment** and reflect on what you’re configuring.

---

## ✅ Goals
By the end of this section, you will:
- Install and configure basic Kubernetes CLI tools
- Create your first local Kubernetes cluster using `K3D`
- Deploy and inspect your first Pod
- Verify your setup is working for the labs ahead

---

## 🛠️ Step-by-Step Setup Instructions

### Step 1: Install `kubectl`
Follow the official guide:  
➡️ https://kubernetes.io/docs/tasks/tools/install-kubectl/

Verify:
```bash
kubectl version --client
```

### Step 2: Install `K3D`
Follow the official guide:  
➡️ https://k3d.io/stable/#installation

Verify:
```bash
k3d version
```

---

### Step 3: Create a Local Cluster

For these labs, I created a cluster from a config file, but you can also create one with a simple command:

```bash
k3d cluster create grow-cluster
```

To create a cluster with a config file, use:

```bash
k3d cluster create --config k3d-config.yaml
```

This cluster has one control-plane node and two worker nodes, as defined in the `k3d-config.yaml` file. In K3D, they are called `server` and `agent` nodes, respectively.

> **Note**: Here I had to enforce the Kubernetes version, because if I simply created a cluster without specifying the version, I got Kubernetes 1.31, which was not compatible with Flux. When running `flux check --pre`, the check didn't pass because of the Kubernetes version. Now it works.

Check your nodes:

```bash
kubectl get nodes
```

---

### Step 4: Deploy Your First Pod
```bash
kubectl run nginx --image=nginx
kubectl get pods
```

Describe it:
```bash
kubectl describe pod nginx
```

Delete it:
```bash
kubectl delete pod nginx
```

---

## 🎯 Next Step

When ready, move on to your first official lab:  
👉 [Kubernetes Fundamentals – Deploying a Pod and Deployment](../01-kubernetes-fundamentals/lab-guide.md)
