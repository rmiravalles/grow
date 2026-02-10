# StatefulSets

StatefulSets in Kubernetes are a workload API object designed for applications that require stable, unique network identifiers, persistent storage, and ordered, controlled deployment/scaling. They are essential when running stateful or distributed systems where each instance (pod) must maintain identity and data across restarts.

## What Is a StatefulSet?

A StatefulSet manages the deployment and scaling of a set of Pods while guaranteeing:

1. Stable, Unique Pod Identity

Each Pod gets a **predictable name** based on its ordinal index:

```bash
<statefulset-name>-0
<statefulset-name>-1
<statefulset-name>-2
```

These names remain consistent even if the Pod is rescheduled.

2. Persistent Volumes per Pod

Each Pod can automatically receive its own **PersistentVolumeClaim (PVC)** via `volumeClaimTemplates`.

These PVCs are **never deleted automatically**, ensuring data persists through restarts.

3. Ordered Deployment, Scaling, and Deletion

StatefulSet ensures:

- Pods start in order (0 → 1 → 2)
- Pods stop in reverse order (2 → 1 → 0)

**Useful when cluster membership matters (e.g., leader election, quorum)**

## Typical Production Use Cases

### Databases

Because databases require persistent storage and predictable addressing:

- PostgreSQL / MySQL operators
- Cassandra
- MongoDB
- Redis (clusters or sentinel)

### Distributed Systems

Systems that rely on stable identities or membership:

- Kafka
- ZooKeeper
- Elasticsearch / OpenSearch
  
### Anything Stateful

Applications that cannot simply “scale out” with stateless replicas:

- Work queues with worker ordering requirements
- Apps that store state locally (e.g., caching layers)

## How StatefulSets Are Used in Production

### With Headless Services for Stable DNS

StatefulSets typically use a headless service:

```yaml
serviceName: my-app
```

This gives each pod a **stable DNS entry**:

```
my-app-0.my-app.default.svc.cluster.local
```

### With VolumeClaimTemplates for Storage

The StatefulSet automatically creates one PVC per Pod:

```yaml
volumeClaimTemplates:
- metadata:
    name: data
  spec:
    storageClassName: gp2
    accessModes: ["ReadWriteOnce"]
    resources:
      requests:
        storage: 20Gi
```

### With Pod Management Policies

You can choose ordered or parallel Pod creation:

- **OrderedReady (default)**: Pods are created, deleted, and updated in a strict sequential order. A Pod must be fully running and ready before the next Pod is created or updated.
- **Parallel**: Pods are created, deleted, and updated simultaneously without waiting for other Pods to become ready.

### With Anti-Affinity for High Availability

To run Pods on separate nodes:

```yml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: my-db
      topologyKey: kubernetes.io/hostname
```

This prevents multiple replicas of a stateful service from landing on the same node.

### With Readiness Probes to Manage Cluster Health

Ensures pods aren’t considered ready until the app has joined a cluster correctly.

### With Operators and CRDs

In production, StatefulSets are often managed by operators, which automate:

- ring formation
- leader election
- shard placement
- storage lifecycle

Examples:

- MongoDB Operator
- Strimzi for Kafka
- Elastic Operator

## Example Minimal StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web"
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

## Summary

StatefulSets solve three key problems for stateful apps:

| Problem | StatefulSet Solution |
|---|---|
| Pod identity | Stable pod names and DNS |
| Pod storage | Persistent volume per Pod |
| Startup ordering | Ordered deployment/scaling |

**Used for**: databases, distributed systems, message queues, and any system needing persistence or stable identity.

# Choosing the Right Pod Management Policy

When deciding between `OrderedReady` and `Parallel` Pod management policies, consider the following factors:

1. Application Requirements: Determine if your application needs a specific order for startup and shutdown. If it does, OrderedReady is the better choice.
2. Scalability Needs: If your application requires rapid scaling and can handle simultaneous Pod operations, Parallel may be more suitable.
3. Complexity of Management: OrderedReady can simplify management for stateful applications, while Parallel may require additional considerations for handling dependencies between Pods.
4. Performance: Parallel can improve deployment speed, but may introduce challenges in ensuring all Pods are ready before serving traffic.
5. Testing: Always test your chosen policy in a staging environment to ensure it meets your application's needs before deploying to production.

## Example Usage

To specify a Pod management policy in a StatefulSet, you can set the `podManagementPolicy` field in the StatefulSet manifest. Here’s an example of a StatefulSet using the OrderedReady policy.

>**Production tip**: Use OrderedReady for stateful systems (Postgres, ZooKeeper, etc.) and switch to Parallel for stateless workloads that only need stable identities.

## 🧪 Commands I Practiced

```bash
kubectl apply -f statefulset.yaml
kubectl get pods -l app=nginx -w
kubectl delete pod web-1 --grace-period=0 --force
kubectl describe statefulset web | tail -20
kubectl delete statefulset web
```