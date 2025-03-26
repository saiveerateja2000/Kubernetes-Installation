## **Approach: Structuring NFS Storage for Multiple Services**

### **1. Create separate directories for each service on the NFS server**
```bash
sudo mkdir -p /mnt/nfs_postgres
sudo mkdir -p /mnt/nfs_redis
sudo mkdir -p /mnt/nfs_nginx
sudo mkdir -p /mnt/nfs_webapp

sudo chmod 777 /mnt/nfs_*  # Open permissions for testing
sudo chown nobody:nogroup /mnt/nfs_*
```

### **2. Add these directories to the NFS exports file**
Edit `/etc/exports` and add:
```
/mnt/nfs_postgres  172.31.12.235(rw,sync,no_subtree_check,no_root_squash)
/mnt/nfs_redis     172.31.12.235(rw,sync,no_subtree_check,no_root_squash)
/mnt/nfs_nginx     172.31.12.235(rw,sync,no_subtree_check,no_root_squash)
/mnt/nfs_webapp    172.31.12.235(rw,sync,no_subtree_check,no_root_squash)
```

### **3. Apply NFS changes**
```bash
sudo exportfs -rav
sudo systemctl restart nfs-kernel-server
```

---

## **Kubernetes Configurations**
For each service, create a **PersistentVolume (PV) and PersistentVolumeClaim (PVC).**

### **PostgreSQL PV & PVC**
#### **PV (pv-postgres.yaml)**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-postgres
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /mnt/nfs_postgres
    server: 172.31.12.235
  persistentVolumeReclaimPolicy: Retain
```

#### **PVC (pvc-postgres.yaml)**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-postgres
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```

---

### **Redis PV & PVC**
#### **PV (pv-redis.yaml)**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-redis
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /mnt/nfs_redis
    server: 172.31.12.235
  persistentVolumeReclaimPolicy: Retain
```

#### **PVC (pvc-redis.yaml)**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-redis
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
```

---

### **Nginx PV & PVC**
#### **PV (pv-nginx.yaml)**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-nginx
spec:
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /mnt/nfs_nginx
    server: 172.31.12.235
  persistentVolumeReclaimPolicy: Retain
```

#### **PVC (pvc-nginx.yaml)**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-nginx
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
```

---

### **Web App PV & PVC**
#### **PV (pv-webapp.yaml)**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-webapp
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /mnt/nfs_webapp
    server: 172.31.12.235
  persistentVolumeReclaimPolicy: Retain
```

#### **PVC (pvc-webapp.yaml)**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-webapp
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```

---

## **Deployment Configuration Example (PostgreSQL)**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:latest
          env:
            - name: POSTGRES_USER
              value: "admin"
            - name: POSTGRES_PASSWORD
              value: "password"
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: nfs-storage
      volumes:
        - name: nfs-storage
          persistentVolumeClaim:
            claimName: nfs-pvc-postgres
```

---

## **Verification Steps**
1. **Apply the PVs and PVCs**
   ```bash
   kubectl apply -f pv-postgres.yaml
   kubectl apply -f pvc-postgres.yaml
   kubectl apply -f pv-redis.yaml
   kubectl apply -f pvc-redis.yaml
   kubectl apply -f pv-nginx.yaml
   kubectl apply -f pvc-nginx.yaml
   kubectl apply -f pv-webapp.yaml
   kubectl apply -f pvc-webapp.yaml
   ```

2. **Check if PVs and PVCs are bound**
   ```bash
   kubectl get pv
   kubectl get pvc
   ```

3. **Deploy the services and check logs**
   ```bash
   kubectl apply -f deployment-postgres.yaml
   kubectl apply -f deployment-redis.yaml
   kubectl apply -f deployment-nginx.yaml
   kubectl apply -f deployment-webapp.yaml
   ```

4. **Check pod storage**
   ```bash
   kubectl exec -it <postgres-pod-name> -- ls /var/lib/postgresql/data
   ```

---

## **Comments on This Approach**
✅ **Isolation:** Each service gets a dedicated NFS directory (`/mnt/nfs_postgres`, `/mnt/nfs_redis`, etc.).  
✅ **Scalability:** If a new service needs storage, just create a new directory, export it in NFS, and set up PV/PVC.  
✅ **Flexibility:** Allows different storage sizes based on service needs.  
✅ **Persistence:** Data remains even if pods restart.  

---

**Let me know if you need modifications! 🚀**

