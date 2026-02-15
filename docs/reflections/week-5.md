# Week 5 Reflection - Use PersistentVolumeClaims to Run a Stateful App

I left [Week 4](week-4.md) aside for a moment, because it was giving me lots of headaches.

This one, however, gave me a lot of joy!

I made some adaptations to the original lab. Instead of deploying a PostgreSQL instance, I deployed MongoDB. Again, just because I'm more familiarized with it, and also because I really like it.

## Storage in Kubernetes

By default, when a Pod in Kubernetes is deleted or restarts, all its data is lost. Pods don't have a way to persist storage on their own, since they aren't built to last.

**PersistentVolumeClaims (PVC)** provide a solution for this problem. They provide permanent storage that survives even when Pods are deleted or restarted.

A very common use for `PVCs` is to permanently store data in Databases, which is the goal of this lab. Your data will survive Pod restarts and will be safe from harm.

### Where does the storage space come from?

This is a question I asked myself, and most probably you too. In the `PersistentVolumeClaim` manifest, you'll determine how much space you'll need, like this:

```yml
resources:
    requests:
      storage: 5Gi
```
But from where the `PVC` is claiming this 5GB of storage that it needs?

The storage comes from **PersistentVolumes (PVs)**, which represent **actual physical or cloud storage**. This is how it works:

```text
PersistentVolume (PV) = The actual storage
       ↓ binds to
PersistentVolumeClaim (PVC) = Your request for storage
       ↓ used by
Pod = Your application
```

### Storage must exist first

Before you can claim storage, someone needs to provide the actual storage. You can do it the manual way or the automatic, more common way.

In the manual way, you'll define your storage specs in a YAML manifest.

```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  hostPath:
    path: /mnt/data  # Actual location on the server
```
You then bind this `PV` resource to your `PVC`, by mentioning it under `spec`.

```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: "" # Empty string must be explicitly set otherwise default StorageClass will be set
  volumeName: my-pv
```

**Automatic Way:** Kubernetes uses a **StorageClass** with a **provisioner** that automatically creates PVs when you create PVCs.

If you're using Kind, you can see where the data is stored in you machine, by inspecting the container running the nodes.

```bash
docker inspect dev-cluster-control-plane
```

This is the name of my control plane.

Look for `volume`.

---

## Using secrets

To add some security, I decided to add a `Secret` object, to store the Database credentials. I created the Secret object, as described in the file below.

```yml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque
stringData:
  # Default credentials (base64 encoded)
  username: admin
  password: password123
```

I learned that these secrets are simply encoded, and anybody can decoded it. These secrets aren't encrypted. In the end, we can't rely on base64 encoding for protection.

## The PVC

This is the file I used to create the PVC.

```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  # storageClassName: standard  # Uncomment if you need to specify a StorageClass
```

To inspect the newly created, I ran the `kubectl describe` command.

```bash
kubectl describe pvc mongodb-pvc
```

![k describe pvc](../assets/describepvc.png)

The `status` is set to `bound`, which means the `PVC` is correctly attached to the actual storage.

## The ConfigMap

Here I used a `ConfigMap` with an initialization script to load the Database with sample data.

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-init-script
data:
  init-mongo.js: |
    // This script runs automatically when MongoDB starts for the first time
    // It creates a sample database with collections and data
    
    db = db.getSiblingDB('sampledb');
    
    // Create a users collection with sample data
    db.users.insertMany([
      {
        name: "John Doe",
        email: "john.doe@example.com",
        age: 30,
        role: "developer",
        created_at: new Date()
      },
      {
        name: "Jane Smith",
        email: "jane.smith@example.com",
        age: 28,
        role: "designer",
        created_at: new Date()
      },
      {
        name: "Bob Johnson",
        email: "bob.johnson@example.com",
        age: 35,
        role: "manager",
        created_at: new Date()
      },
      {
        name: "Sandy Saunders",
        email: "sandy.saunders@example.com",
        age: 27,
        role: "DB Admin",
        created_at: new Date()
      }
    ]);
    
    // Create a products collection with sample data
    db.products.insertMany([
      {
        name: "Laptop",
        category: "Electronics",
        price: 999.99,
        stock: 50,
        description: "High-performance laptop for professionals",
        created_at: new Date()
      },
      {
        name: "Wireless Mouse",
        category: "Electronics",
        price: 29.99,
        stock: 200,
        description: "Ergonomic wireless mouse",
        created_at: new Date()
      },
      {
        name: "Office Chair",
        category: "Furniture",
        price: 299.99,
        stock: 30,
        description: "Comfortable ergonomic office chair",
        created_at: new Date()
      },
      {
        name: "Desk Lamp",
        category: "Furniture",
        price: 49.99,
        stock: 100,
        description: "LED desk lamp with adjustable brightness",
        created_at: new Date()
      },
      {
        name: "Sweatshirt",
        category: "Apparel",
        price: 23.49,
        stock: 70,
        description: "Long-sleeved high quality cotton garment",
        created_at: new Date()
      }
    ]);
    
    // Create a orders collection with sample data
    db.orders.insertMany([
      {
        user_email: "john.doe@example.com",
        products: [
          { name: "Laptop", quantity: 1, price: 999.99 },
          { name: "Wireless Mouse", quantity: 2, price: 29.99 }
        ],
        total: 1059.97,
        status: "completed",
        order_date: new Date()
      },
      {
        user_email: "jane.smith@example.com",
        products: [
          { name: "Office Chair", quantity: 1, price: 299.99 },
          { name: "Desk Lamp", quantity: 1, price: 49.99 }
        ],
        total: 349.98,
        status: "pending",
        order_date: new Date()
      }
    ]);
    
    // Create indexes for better query performance
    db.users.createIndex({ email: 1 }, { unique: true });
    db.products.createIndex({ category: 1 });
    db.orders.createIndex({ user_email: 1 });
    
    print('Sample database initialized successfully!');
    print('Collections created: users, products, orders');
    print('Total users: ' + db.users.countDocuments());
    print('Total products: ' + db.products.countDocuments());
    print('Total orders: ' + db.orders.countDocuments());
```

## The deployment

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
  labels:
    app: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:latest
        ports:
        - containerPort: 27017
          name: mongodb
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: password
        - name: MONGO_INITDB_DATABASE
          value: sampledb
        volumeMounts:
        - name: mongodb-data
          mountPath: /data/db
        - name: mongodb-init
          mountPath: /docker-entrypoint-initdb.d
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
      volumes:
      - name: mongodb-data
        persistentVolumeClaim:
          claimName: mongodb-pvc
      - name: mongodb-init
        configMap:
          name: mongodb-init-script
```

## The service

```yml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-nodeport
  labels:
    app: mongodb
spec:
  type: NodePort
  ports:
  - port: 27017
    targetPort: 27017
    nodePort: 30017
    protocol: TCP
    name: mongodb
  selector:
    app: mongodb
```

## Making changes to the Database

To check if the `PVC` was working and storing the data properly, I added an entry to the Database and then killed the Pod. The `ReplicaSet` kicked in and created a new Pod.

### Adding a new entry to the Database

For this task, I used [MongoDB Compass](https://www.mongodb.com/try/download/compass), a super handy GUI.

I first port-forwarded the service, so I could connect to it.

```bash
kubectl port-forward service/mongodb-nodeport 27017:27017
```

Then, from the browser, I navigated to this address: `mongodb://admin:password123@localhost:27017/sampledb?authSource=admin`

A pop-up window prompted me to open Compass.

I added Mr. Jimmy James to the list of users. This is how it looks like.

![compass](../assets/compass.png)

After the Pod restarted, I used the [MongoDB Shell](https://www.mongodb.com/try/download/shell) to check the contents of my Database.

From my terminal, I ran this command.

```bash
kubectl exec -it deployment/mongodb -- mongosh -u $(MONGODB_USERNAME) -p $(MONGODB_PASSWORD) --authenticationDatabase admin
```

![shell](../assets/mongodbshell.png)

And there is Mr. Jimmy James!

## But there's more!

I want to go back here and add more reflections.










