Lab 3: Horizontal Pod Autoscaler

command to install metric server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

Task 1: Creating a Deployment and Horizontal Pod Autoscaler

vi hpa-deploy.yaml

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


kubectl create -f hpa-deploy.yaml
kubectl get deployments
kubectl top pods
kubectl autoscale deployment nginx-deployment --min=3 --max=6 --cpu-percent=50
kubectl get hpa
kubectl describe hpa nginx-deployment
kubectl exec -it <pod_name> -- bash
# while true; do true; done
kubectl get hpa nginx-deployment
kubectl get deployments nginx-deployment
kubectl describe hpa nginx-deployment
kubectl delete hpa nginx-deployment
kubectl delete deployments nginx-deployment
