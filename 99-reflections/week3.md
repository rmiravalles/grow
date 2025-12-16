# Week 3 Reflection – Kubernetes Security

This week, I worked through the Kubernetes security lab focused on applying `securityContext` settings and exploring pod-level hardening techniques. I also learned about optional security admission and runtime tools like Kyverno and Falco.


---

## ✅ What I Learned

### What happens if a container runs as root?

If a container runs as root, it means the main process inside the container has **UID 0**. What that implies depends on how the container is configured and where it’s running, but there are important security consequences.

- Inside the container, root has full control of the container's filesystem and processes.
- By default, root inside the container = root on the host for many kernel interactions.
- **Containers share the host kernel, so this is very different from a VM**.

If an attacker gains root inside a container and:

- There is a kernel vulnerability
- Capabilities are overly permissive
- The container is run with `--privileged`
- Host files are mounted (e.g., `/var/run/docker.sock`, `/`, `/proc`)

➡️ The attacker may escape the container and gain host root.

If the container can access `var/run/docker.sock`, root inside the container can:

- Start new containers with host-level privileges
- Mount host filesystems
- Gain full control of the host

### What if an attacker escapes the container namespace?

If an attacker escapes the container namespace, they can potentially access the host system directly. This could lead to:

- Gaining root access to the host
- Accessing sensitive files on the host
- Compromising other containers running on the same host
- Moving laterally within the network to attack other systems

### What can be done to restrict file system access?

To restrict file system access, several strategies can be employed:

- Use `readOnlyRootFilesystem: true` in the Pod's security context to prevent write access to the root filesystem.
- Avoid mounting sensitive host directories into containers.
- Use Kubernetes `SecurityContext` settings to drop unnecessary capabilities.
- Implement Pod Security Policies or Pod Security Admission to enforce security standards.

### Key SecurityContext Settings Applied

- `runAsNonRoot: true`: Forces the container to **refuse startup if it tries to run as root (UID 0)**.
- `runAsUser: 1000`: Specifies the user ID to run the container process as a non-root user. UID 1000 is a common choice for non-root users.
  
> If `runAsUser` is set to 0, the Pod will fail to start because it violates the `runAsNonRoot` policy.

- `allowPrivilegeEscalation: false`: Prevents processes from gaining additional privileges. This blocks `sudo`, and stops actions like `setuid` or `setgid` that could elevate privileges.
- `readOnlyRootFilesystem: true`: Mounts the container's **root filesystem as read-only**. It prevents any write operations to locations like `/`,`/etc`, `/bin`, `/usr`. Apps needing write access must use specific writable volumes.
- `capabilities.drop: [ALL]`: Removes **all Linux capabilities** to minimize the attack surface.
 
### Pod Security Admission

The Pod Security Admission (PSA) Controller is a built-in Kubernetes admission controller (GA since Kubernetes v1.25) that enforces Pod-level security standards at namespace boundaries. It replaces the deprecated PodSecurityPolicy (PSP) mechanism with a simpler, more predictable, and label-based model. The Pod Security Admission Controller ensures that Pods comply with predefined security profiles before they are admitted into the cluster.


---

## ❓ What Was Challenging

-
- 
- 
- 
-  

---

## 🧪 Commands I Practiced

```bash




```

---

## 🔐 Security Awareness Gained

-
- 
- 
- 
 

---

## 🧠 Real-World Relevance

How would I use these security settings in production?



---

## 📎 Related Files

- `secure-pod.yaml`
- Namespace label configuration for PodSecurity
- Notes on failed pod creation due to missing `runAsNonRoot`

---

## 📝 Questions I Still Have

-
-
-

---


## 🧠 Bonus Reflection

Try this:  
If I had to defend a production cluster, what would I lock down first?

---

## 🚀 What's Next

I’m considering submitting a custom Week 4 security lab idea—maybe one that combines Kyverno with GitOps. 
