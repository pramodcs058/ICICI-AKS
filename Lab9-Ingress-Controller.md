# **Lab: Configuring Ingress Controller in AKS Kubernetes Cluster**

## **Prerequisites**
1. An AKS cluster is already set up.
2. `kubectl` is installed and configured to access the AKS cluster.
3. Helm is installed (optional for ingress controller setup).
4. Basic knowledge of Kubernetes objects.

---

---

## **Step 2: Deploy Two Pods**

### **2.1 Create NGINX Pod and ClusterIP Service**

**Pod YAML (nginx-pod.yaml):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx-container
    image: nginx
    ports:
    - containerPort: 80
```

**ClusterIP Service YAML (nginx-svc.yaml):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

Apply both YAML files:  
```bash
kubectl apply -f nginx-pod.yaml
kubectl apply -f nginx-svc.yaml
```

### **2.2 Create HTTPD Pod and ClusterIP Service**

**Pod YAML (httpd-pod.yaml):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: httpd
  labels:
    app: httpd
spec:
  containers:
  - name: httpd-container
    image: httpd
    ports:
    - containerPort: 80
```

**ClusterIP Service YAML (httpd-svc.yaml):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: httpd-svc
spec:
  selector:
    app: httpd
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

Apply both YAML files:  
```bash
kubectl apply -f httpd-pod.yaml
kubectl apply -f httpd-svc.yaml
```

Verify the pods and services are running:  
```bash
kubectl get pods
kubectl get svc
```

---

---

## **Step 3: Create Ingress Resource**

**Ingress YAML (ingress.yaml):**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test
  namespace: default
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    appgw.ingress.kubernetes.io/backend-path-prefix: /  
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-svc
                port: 
                  number: 80
    - http:
        paths:
          - path: /httpd
            pathType: Prefix
            backend:
              service:
                name: httpd-svc
                port: 
                  number: 80
```

Apply the ingress configuration:  
```bash
kubectl apply -f ingress.yaml
```

---

## **Step 4: Test Ingress**

### **Retrieve the Ingress Controller’s External IP**
```bash
kubectl get svc -n ingress-nginx
```

Find the `EXTERNAL-IP` of the ingress controller (e.g., `LoadBalancer` service). Use this IP for testing.

### **Test the Default Path (NGINX)**
Open a browser or use `curl` to test:  
```bash
curl http://<EXTERNAL-IP>/
```

### **Test the HTTPD Path**
```bash
curl http://<EXTERNAL-IP>/httpd
```

You should see the default pages served by NGINX and HTTPD.

---

## **Step 5: Cleanup**

After the lab, clean up the resources:  
```bash
kubectl delete namespace ingress-lab
```

