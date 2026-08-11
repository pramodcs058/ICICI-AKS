# Lab: Services in Kubernetes on AKS

## Objective
This lab will guide you through creating a Kubernetes pod, configuring various service types (ClusterIP, NodePort, and LoadBalancer), and managing the lifecycle of pods and services in an AKS cluster.

---

## Prerequisites
1. An AKS cluster set up and running. Refer to the [Lab for Setting Up AKS](#) if required.
2. Access to the AKS cluster using `kubectl`.
3. Basic knowledge of YAML syntax and Kubernetes concepts.

---

## Tasks Overview
1. Create a pod.
2. Configure and validate a ClusterIP service.
3. Modify and validate a NodePort service.
4. Configure and validate a LoadBalancer service.
5. Manage pod lifecycle and observe service behavior.
6. Cleanup resources.

---

### Task 1: Create a Pod

#### Create the YAML for the Pod:
```bash
vi httpd-pod.yaml
```

#### Add the following configuration:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: httpd-pod
  labels:
    env: prod
    type: front-end
    app: httpd-ws
spec:
  containers:
  - name: httpd-container
    image: httpd
    ports:
    - containerPort: 80
```

#### Apply the Pod YAML:
```bash
kubectl create -f httpd-pod.yaml
```

#### Verify the Pod:
```bash
kubectl get pods
kubectl get pods -o wide
kubectl describe pod httpd-pod
```

---

### Task 2: Setup ClusterIP Service

#### Create the YAML for the ClusterIP Service:
```bash
vi httpd-svc.yaml
```

#### Add the following configuration:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: httpd-svc
spec:
  selector:
    env: prod
    type: front-end
    app: httpd-ws
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

#### Apply the Service YAML:
```bash
kubectl apply -f httpd-svc.yaml
```

#### Validate the Service:
```bash
kubectl get svc
kubectl describe svc httpd-svc
kubectl get ep
```
---
### Task 3: Modify Service to NodePort

#### Update the Service YAML:
```bash
vi httpd-svc.yaml
```

#### Modify the `type` to `NodePort`:
```yaml
type: NodePort
```

#### Apply the Changes:
```bash
kubectl apply -f httpd-svc.yaml
```

#### Validate the NodePort Service:
```bash
kubectl describe svc httpd-svc
```
---
### Task 4: Modify Service to LoadBalancer

#### Update the Service YAML:
```bash
vi httpd-svc.yaml
```

#### Modify the `type` to `LoadBalancer`:
```yaml
type: LoadBalancer
```

#### Apply the Changes:
```bash
kubectl apply -f httpd-svc.yaml
```

#### Verify the LoadBalancer Service:
```bash
kubectl get svc
kubectl describe svc httpd-svc
```
---
### Task 5: Delete and Recreate Pod

#### Delete the Existing Pod:
```bash
kubectl delete -f httpd-pod.yaml
```

#### Observe the Service:
```bash
kubectl describe svc httpd-svc
```

#### Recreate the Pod:
```bash
kubectl apply -f httpd-pod.yaml
kubectl describe svc httpd-svc
```

---

### Task 6: Cleanup Resources

#### Delete the Resources:
```bash
kubectl delete -f httpd-pod.yaml
kubectl delete -f httpd-svc.yaml
```
