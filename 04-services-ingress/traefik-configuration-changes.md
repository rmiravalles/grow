# Traefik Ingress Configuration Changes for Kind

## Problem Statement

When running `kubectl get ingress`, the ADDRESS field was empty, and attempting to access the ingress resulted in a connection reset error:

```bash
$ kubectl get ingress
NAME           CLASS     HOSTS              ADDRESS   PORTS   AGE
demo-ingress   traefik   demo.example.com             80      13h

$ curl -H "Host: demo.example.com" http://localhost
curl: (56) Recv failure: Connection reset by peer
```

## Root Cause Analysis

### Issue 1: Incorrect Pod Placement
The Traefik pod was scheduled on a worker node (`dev-cluster-worker`) instead of the control-plane node. In Kind clusters, only the control-plane node has the `extraPortMappings` configured in [kind-config.yaml](kind-config.yaml):

```yaml
extraPortMappings:
- containerPort: 80
  hostPort: 80
  protocol: TCP
- containerPort: 443
  hostPort: 443
  protocol: TCP
```

This meant traffic from `localhost:80` couldn't reach Traefik running on a worker node.

### Issue 2: Incorrect Service Type
The Traefik service was configured as `LoadBalancer` with NodePort settings:
- LoadBalancer services don't work in Kind without MetalLB
- NodePort (30080/30443) wasn't accessible due to port mapping issues
- The correct approach for Kind is to use `hostPort` with the control-plane node

### Issue 3: Empty ADDRESS Field
The empty ADDRESS field was a symptom of using LoadBalancer service type in a local cluster without a cloud provider or MetalLB installation.

## Changes Made

### File: `traefik-values.yaml`

#### Before:
```yaml
service:
  type: NodePort
ports:
  web:
    nodePort: 30080
    exposedPort: 80
  websecure:
    nodePort: 30443
    exposedPort: 443
```

#### After:
```yaml
service:
  type: ClusterIP

ports:
  web:
    exposedPort: 80
    hostPort: 80
  websecure:
    exposedPort: 443
    hostPort: 443

# Schedule Traefik on the control-plane node (required for Kind)
nodeSelector:
  ingress-ready: "true"

# Allow scheduling on control-plane (has NoSchedule taint)
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Equal
    effect: NoSchedule
  - key: node-role.kubernetes.io/master
    operator: Equal
    effect: NoSchedule
```

## Detailed Change Descriptions

### 1. Service Type: `LoadBalancer/NodePort` → `ClusterIP`

**What changed:** Changed the service type from LoadBalancer to ClusterIP.

**Why:** 
- LoadBalancer services require an external load balancer provider (e.g., cloud provider or MetalLB)
- In Kind, we use `hostPort` to bind directly to the host's network interface
- ClusterIP is sufficient when using hostPort for pod-level port binding

**Impact:** Service is now accessible via hostPort binding instead of LoadBalancer or NodePort.

### 2. Port Configuration: Added `hostPort`

**What changed:** 
- Removed `nodePort: 30080` and `nodePort: 30443`
- Added `hostPort: 80` and `hostPort: 443`

**Why:**
- `hostPort` binds the pod's port directly to the node's network interface
- This works with Kind's `extraPortMappings`, which forwards `localhost:80` → `control-plane:80`
- The traffic flow becomes: `localhost:80` → `control-plane:80` → `Traefik pod:80`

**Impact:** Traefik is now accessible on `localhost:80` and `localhost:443` as configured in kind-config.yaml.

### 3. Added `nodeSelector`

**What changed:** Added nodeSelector to target nodes with label `ingress-ready: "true"`.

```yaml
nodeSelector:
  ingress-ready: "true"
```

**Why:**
- The Kind cluster configuration sets this label on the control-plane node
- The control-plane node is the only node with extraPortMappings configured
- This ensures Traefik pods are scheduled only on nodes that have host port forwarding

**Impact:** Traefik pods now consistently run on the control-plane node where port 80/443 are accessible.

### 4. Added `tolerations`

**What changed:** Added tolerations for control-plane taints.

```yaml
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Equal
    effect: NoSchedule
  - key: node-role.kubernetes.io/master
    operator: Equal
    effect: NoSchedule
```

**Why:**
- Kubernetes control-plane nodes typically have a `NoSchedule` taint to prevent regular workloads from running on them
- Without tolerations, the scheduler would refuse to place Traefik on the control-plane
- The `node-role.kubernetes.io/master` is for backward compatibility with older clusters

**Impact:** Traefik can now be scheduled on the control-plane node despite its taints.

## Deployment Commands

After making the configuration changes, the following commands were executed:

```bash
# Apply the updated configuration
helm upgrade traefik traefik/traefik -n traefik -f traefik-values.yaml

# Wait for rollout to complete
kubectl rollout status deployment traefik -n traefik

# Verify pod placement
kubectl get pod -n traefik -o wide
```

## Verification

### Pod Placement Verification
```bash
$ kubectl get pod -n traefik -o wide
NAME                       READY   STATUS    RESTARTS   AGE   NODE
traefik-7c8f6698d9-jkk4w   1/1     Running   0          20s   dev-cluster-control-plane
```
✅ Traefik is now running on the control-plane node.

### Service Type Verification
```bash
$ kubectl get svc -n traefik
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
traefik   ClusterIP   10.96.146.61   <none>        80/TCP,443/TCP   14h
```
✅ Service is now ClusterIP type.

### Ingress Functionality Test
```bash
$ curl -H "Host: demo.example.com" http://localhost
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
```
✅ Ingress is now working correctly and returns the expected response.

### Ingress Status
```bash
$ kubectl get ingress
NAME           CLASS     HOSTS              ADDRESS   PORTS   AGE
demo-ingress   traefik   demo.example.com             80      14h
```
⚠️ The ADDRESS field remains empty - this is **expected and correct** for Kind clusters with hostPort configuration.

## Understanding the Empty ADDRESS Field

The empty ADDRESS field is **normal and expected** for Kind clusters using hostPort. Here's why:

1. **ADDRESS field purpose:** Displays the external IP/hostname where the ingress is accessible
2. **LoadBalancer behavior:** Would show the external IP assigned by the load balancer
3. **Kind with hostPort:** No external IP is assigned; traffic goes through localhost via port forwarding
4. **Not an error:** The ingress works correctly despite the empty field

The actual access path is:
```
Client (localhost:80) 
  → Kind extraPortMappings 
    → Control-plane node (hostPort 80) 
      → Traefik pod 
        → Backend service (demo:80)
```

## Key Takeaways

1. **For Kind clusters:** Use `hostPort` with `ClusterIP` service, not `LoadBalancer` or `NodePort`
2. **Pod placement matters:** Traefik must run on the node with extraPortMappings (control-plane)
3. **Tolerations are required:** Control-plane nodes have NoSchedule taints by default
4. **Empty ADDRESS is OK:** In local clusters, this field may remain empty while ingress still works
5. **Test with Host header:** Always include the correct Host header when testing ingress locally

## Additional Resources

- [Kind Ingress Documentation](https://kind.sigs.k8s.io/docs/user/ingress/)
- [Traefik Kubernetes Ingress](https://doc.traefik.io/traefik/routing/providers/kubernetes-ingress/)
- [Kubernetes Taints and Tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/)
