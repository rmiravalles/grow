# Week 1 Reflection – NGINX Pod and Deployment

This week, I worked through the Kubernetes fundamentals lab where I deployed a basic NGINX web server using both a Pod and a Deployment. I also learned about security contexts and how to scale workloads.

---

# ✅ What I Learned

## Pods vs Deployments

**Pods** and **Deployments** are related but serve different purposes in Kubernetes.

### Pods

- A Pod is the smallest deployable unit in Kubernetes.
- It represents one or more containers that share storage and network resources.
- Pods are ephemeral; if a Pod fails, it does not automatically get replaced.
- Pods cannot be scaled directly. We'd need to create new Pods manually or use a higher-level controller.
- Pods can be used for simple, single-instance applications and short-lived tasks.

### Deployments

- A **controller** that manages **ReplicaSets**, which in turn manage Pods.
- Ensure that the **desired state** (e.g., number of replicas of a Pod) is continuously maintained.
- Support rolling updates and rollbacks.
- Allow for easy scaling of applications by adjusting the number of replicas.

## Security Contexts

- Security contexts define privilege and access control settings for Pods or containers.
- Key settings include:
  - `runAsNonRoot`: Ensures the container does not run as the root user.
  - `allowPrivilegeEscalation`: Prevents processes from gaining more privileges than their parent process.
- Implementing security contexts enhances the security posture of workloads by minimizing potential attack vectors.

When `runAsNonRoot: true` is set, Kubernetes checks the UID the container will run with.

The container must run as a **non-zero UID** (i.e., not UID 0).

If the image’s default user is root (UID 0) and you have not explicitly set a different user (via `runAsUser` or in the Dockerfile), Kubernetes will prevent the Pod from starting.
  
---

## ❓ What Was Challenging

- 
- 
- 

---

## 🧪 Commands I Practiced

```bash




```

---

## 🔐 Security Improvements I Made

- 
- 

---

## 📝 Questions I Still Have

- 
- 
- 

---

## 📎 Related YAMLs

- `nginx-pod.yaml`
- `nginx-deployment.yaml`

---

## 🚀 Looking Ahead

I’m excited to explore GitOps in Week 2 and automate deployments using FluxCD.