# Azure Gateway Ingress - Health Probes

## Step-01: Introduction
- We are going to understand and implement Health Probes
- We are going to learn that 
- **Usecase-1:** when we use `defaultBackend` in Ingress Service how is the Health probe created
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress-service
spec:
  ingressClassName: azure-application-gateway
  defaultBackend:
    service:
      name: myapp-nginx-clusterip-service
      port: 
        number: 80
```
- **Usecase-2:** When we use `HTTP Paths` in Ingress Service how is the Health probe created
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginxapp1-ingress-service
spec:
  ingressClassName: azure-application-gateway
  rules:
    - http:
        paths:
          - path: /app1
            pathType: Prefix
            backend:
              service:
                name: app1-nginx-clusterip-service
                port: 
                  number: 80
```

## Step-02: Review k8s Application Manifests
### kube-manifests: 01-kube-manifests-default-backend
- 01-myapp-Deployment.yaml
- 02-myapp-ClusterIP-Service.yaml
- 03-Ingress-Default-Backend.yaml
### kube-manifests: 02-kube-manifests-http-path
- 01-NginxApp1-Deployment.yaml
- 02-NginxApp1-ClusterIP-Service.yaml
- 03-Ingress-HTTP-Paths.yaml

## Step-03: Deploy and Verify
```t
# Deploy Apps
kubectl apply -f 01-kube-manifests-default-backend/
kubectl apply -f 02-kube-manifests-http-path/

# List Deployments
kubectl get deploy

# List Pods
kubectl get pods

# List Services
kubectl get svc

# List Ingress
kubectl get ingress

# Access Application (curl)
curl http://<INGRESS-IP>
curl http://<INGRESS-IP>/app1/index.html

# Access Application (Browser)
http://<INGRESS-IP>
http://<INGRESS-IP>/app1/index.html
```

## Step-04: Verify Application Gateway using Azure Portal
- **Backend Settings Tab:** 
  - Review both applications `backend settings`
  - Primarily review `Custom Probe`
  - **Default Backend App:** Should use the `defaultprobe-Http probe` 
  - **HTTP Path App:** This should creata a custom probe with `/app1` as path
- **Health Probes Tab:**   
  - Review both Health probes
  - Perform Health probe **TEST**
```t
# Verify Pods logs during Probe Test
kubectl get pods
kubectl logs -f <POD-NAME>
```  

## Step-05: Clean-Up Applications
```t
# Delete Apps
kubectl delete -f 01-kube-manifests-default-backend/
kubectl delete -f 02-kube-manifests-http-path/

# Review Application Gateway Tabs
1. Backend Pools Tab 
2. Backend Settings Tab
3. Rules Tab
```
