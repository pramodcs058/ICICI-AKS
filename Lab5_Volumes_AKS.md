# 🧪 LAB: Working with Persistent Volumes in AKS (Default StorageClass)

## 🎯 Objective

By the end of this lab, students will:

* Understand **PersistentVolume (PV)** and **PersistentVolumeClaim (PVC)**
* Use **default StorageClass in AKS**
* Deploy an app with **persistent storage**
* Verify **data persistence**

---

# 🧰 Prerequisites

* AKS Cluster created
* kubectl configured
* Verify connection:

```bash
kubectl get nodes
```

---

# 🔍 Step 1: Check Default StorageClass

AKS automatically provides a default StorageClass (usually Azure Disk).

```bash
kubectl get storageclass
```


# 🧱 Step 2: Create a PersistentVolumeClaim (PVC)

👉 No need to create PV manually (dynamic provisioning will happen)

### 📄 pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

Apply it:

```bash
kubectl apply -f pvc.yaml
```

---

# 🔎 Step 3: Verify PVC & PV

```bash
kubectl get pvc
kubectl get pv
```

👉 You will see:

* PVC = Bound
* PV = Automatically created (Azure Disk)

---

# 🚀 Step 4: Create Pod Using PVC

### 📄 pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
  - name: app-container
    image: nginx
    volumeMounts:
    - mountPath: /usr/share/nginx/html
      name: my-volume
  volumes:
  - name: my-volume
    persistentVolumeClaim:
      claimName: demo-pvc
```

Apply:

```bash
kubectl apply -f pod.yaml
```

---

# 🔍 Step 5: Verify Pod & Volume Mount

```bash
kubectl get pods
kubectl describe pod volume-pod
```

👉 Confirm:

* PVC is attached
* Volume mounted successfully

---

# ✍️ Step 6: Write Data to Volume

```bash
kubectl exec -it volume-pod -- /bin/bash
```

Inside container:

```bash
echo "Hello from AKS Volume Lab" > /usr/share/nginx/html/index.html
exit
```

---

# 🌐 Step 7: Expose Pod

```bash
kubectl label pod volume-pod app=frontend
kubectl expose pod volume-pod --type=LoadBalancer --port=80
kubectl get svc
```

👉 Access in browser:

```
http://<NodeIP>:<NodePort>
```

✔️ You should see:
👉 **Hello from AKS Volume Lab**

---

# 🔁 Step 8: Test Persistence

Delete pod:

```bash
kubectl delete pod volume-pod
```

Recreate pod:

```bash
kubectl apply -f pod.yaml
kubectl label pod volume-pod app=frontend
```

---

# ✅ Step 9: Verify Data Still Exists

Access again:

👉 Same message should appear:

```
Hello from AKS Volume Lab
```

✔️ This proves:
👉 Data is stored in **Azure Disk (Persistent Storage)**

---

# 🧹 Step 10: Cleanup

```bash
kubectl delete pod volume-pod
kubectl delete pvc demo-pvc
```



---

If you want next level:

* I can create **Azure Files (RWX) lab**
* Or **StatefulSet with persistent storage**
* Or **real-world project (WordPress + MySQL with volumes)**
