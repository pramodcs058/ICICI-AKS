# **Lab 10: Horizontal Pod Autoscaler**

This lab demonstrates how Kubernetes automatically increases or decreases the number of Pods based on CPU utilization.

---

## **Task 1: Install Metrics Server**

Metrics Server provides CPU and memory metrics required by HPA.

### **Step 1: Install Metrics Server**

Run:

```bash
curl -L https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml -o metrics-server.yaml
```

This installs or updates the Metrics Server components in the cluster.

> **Note:** If your AKS cluster shows a resource validation error while applying this command, contact the trainer before making changes to the Metrics Server configuration.

---

## **Task 2: Create a Deployment and Horizontal Pod Autoscaler**

### **Step 1: Create the Deployment manifest**

Create a file named `hpa-deploy.yaml`:

```bash
vi hpa-deploy.yaml
```

Add the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-deployment
  template:
    metadata:
      labels:
        app: nginx-deployment
    spec:
      containers:
      - image: nginx
        name: nginx
        resources:
          limits:
            cpu: 200m
            memory: 100Mi
          requests:
            cpu: 100m
            memory: 50Mi
```

The Deployment starts three NGINX Pods with CPU and memory requests and limits.

---

### **Step 2: Create the Deployment**

```bash
kubectl create -f hpa-deploy.yaml
```

This creates the NGINX Deployment in the current namespace.

---

### **Step 3: Check the Deployment**

```bash
kubectl get deployments
```

This displays the Deployment and its current replica status.

---

### **Step 4: Check Pod Metrics**

```bash
kubectl top pods
```

This displays the current CPU and memory usage of the Pods.

---

### **Step 5: Create the Horizontal Pod Autoscaler**

```bash
kubectl autoscale deployment nginx-deployment --min=3 --max=6 --cpu-percent=50
```

This creates an HPA that maintains between 3 and 6 Pods with a target CPU utilization of 50%.

---

### **Step 6: Check the HPA**

```bash
kubectl get hpa
```

This displays the HPA configuration, current CPU utilization, and replica count.

---

### **Step 7: Describe the HPA**

```bash
kubectl describe hpa nginx-deployment
```

This displays detailed HPA information, including metrics, conditions, and scaling events.

---

### **Step 8: Open a Shell Inside a Pod**

First identify a Pod using the Pod name from the previous commands.

```bash
kubectl exec -it <pod_name> -- bash
```

This opens a Bash shell inside the selected NGINX Pod.

---

### **Step 9: Generate CPU Load**

Inside the Pod, run:

```bash
while true; do true; done
```

This continuously generates CPU activity to demonstrate HPA scaling.

---

### **Step 10: Check the HPA**

Open another terminal and run:

```bash
kubectl get hpa nginx-deployment
```

This shows the current CPU utilization and the number of replicas managed by HPA.

---

### **Step 11: Check the Deployment**

```bash
kubectl get deployments nginx-deployment
```

This shows whether the number of Deployment replicas has increased.

---

### **Step 12: Describe the HPA**

```bash
kubectl describe hpa nginx-deployment
```

This shows the HPA scaling decisions and related events.

---

## **Task 3: Clean Up the Resources**

### **Step 1: Delete the HPA**

```bash
kubectl delete hpa nginx-deployment
```

This removes the Horizontal Pod Autoscaler.

---

### **Step 2: Delete the Deployment**

```bash
kubectl delete deployments nginx-deployment
```

This removes the NGINX Deployment and its Pods.

---

## **Key Points**

- **Metrics Server** provides resource usage metrics.
- **Deployment** manages the NGINX Pods.
- **HPA** automatically adjusts the number of Pods.
- **Minimum replicas:** 3.
- **Maximum replicas:** 6.
- **CPU target:** 50%.
- **Scale-out:** More Pods are created when CPU utilization increases.
- **Scale-in:** Pods can be reduced when CPU utilization decreases.

---
---

## **End of Lab**
