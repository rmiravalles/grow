# 🚀 Lab: Install and Configure FluxCD for GitOps

Welcome to GitOps! In this lab, you'll learn how to use **FluxCD** to automatically deploy Kubernetes resources from your Git repository.

GitOps treats Git as the single source of truth for your cluster. Changes are made by **committing YAML to a repo**, not running `kubectl apply` manually.

---

## 🎯 Objectives

By the end of this lab, you will:

- Understand GitOps as a deployment model
- Install FluxCD components into your cluster
- Bootstrap a Git repository as a GitOps source
- Create a `Kustomization` that applies YAML from your repo
- Reflect on how GitOps changes your workflow

---

## 🛠 Prerequisites

- A running Kubernetes cluster (`kind`, `k3s`, etc.)
- Flux CLI installed:  
  ➡️ [https://blog.kubeskills.com/hands-on-with-fluxcd](https://blog.kubeskills.com/hands-on-with-fluxcd)
- Your student notebook forked & pushed to GitHub

---

## ✅ Part 1: Bootstrap FluxCD

1. Confirm `flux` is installed:

```bash
flux --version
```
> HINT: Install flux using [this guide](https://blog.kubeskills.com/hands-on-with-fluxcd#heading-installing-fluxcd)

2. Bootstrap your repo:

```bash
flux bootstrap github \
  --owner=rmiravalles \
  --repository=grow \
  --branch=main \
  --path=./clusters/grow \
  --personal
```

This command installs all Flux components and connects your Git repo to your cluster.

---

## ✅ Part 2: Create a GitRepository Source

```bash
flux create source git grow \
  --url=https://github.com/rmiravalles/grow \
  --branch=main \
  --interval=30s
```

Check the source:

```bash
flux get sources git
```

---

## ✅ Part 3: Add a Kustomization

Create a new folder (e.g., `clusters/dev/`) and commit a manifest like this:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: student-notebook
  namespace: flux-system
spec:
  interval: 1m
  path: ./labs
  prune: true
  sourceRef:
    kind: GitRepository
    name: student-notebook
```

Apply it:

```bash
kubectl apply -f clusters/dev/kustomization.yaml
```

Then check the status:

```bash
flux get kustomizations
```

---

## 🧠 Reflect in Your Notebook

Open: `99-reflections/week2.md`

## What I Learned
- How FluxCD keeps a cluster in sync with Git
- Why GitOps reduces manual error and improves auditing

## What Was Challenging
- Understanding how `Kustomization` relates to file paths
- Dealing with errors from missing YAML or invalid patches

## Commands I Practiced
```bash
flux bootstrap github ...
flux create source git ...
flux create kustomization ...
flux get sources
flux get kustomizations
```

## GitOps Impact
- I now track all changes in Git, not in CLI history
- I can rebuild the cluster state by cloning a repo

---

## 📝 Commit Your Configuration

Track your manifests and commit them:

```bash
git add clusters/dev
git commit -m "Add FluxCD Git source and Kustomization"
git push origin main
```

---

## 📣 Share Your Progress

Post your experience using:
- 🏷 `#KubeSkillsGROW`
- 🏷 `#GitOps`

Share your repo or a screenshot of `flux get kustomizations`!

---

## 🔁 Next Step

Ready to add cluster security policies?  
👉 [Continue to Security Labs](../03-security/lab-guide.md)
