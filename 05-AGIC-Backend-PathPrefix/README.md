# Azure Gateway Ingress - Backend Annotations Path Prefix

## Step-01: Introduction
- We are going to understand and implement a Backend Annotation listed below
```yaml
# Azure Ingress Backend Annotation
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway  
    appgw.ingress.kubernetes.io/backend-path-prefix: "/app1/"    
```

## Step-02: Review k8s Application Manifests
- **kube-manifests:** 
- 01-NginxApp1-Deployment.yml
- 02-NginxApp1-ClusterIP-Service.yaml
- 03-Ingress-backend-path-prefix.yml

## Step-03: Review Ingress Service Manifests
- [Ingress API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/)
- [AGIC Backend Hostname Annotation](https://azure.github.io/application-gateway-kubernetes-ingress/annotations/#backend-path-prefix)
- **Ingress Manifest:** 03-Ingress-backend-path-prefix.yml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginxapp1-ingress-service
  annotations:
    appgw.ingress.kubernetes.io/backend-path-prefix: "/app1/"    
spec:
  ingressClassName: azure-application-gateway
  rules:
    - http:
        paths:
          - path: /myapps/app1
            pathType: Prefix
            backend:
              service:
                name: app1-nginx-clusterip-service
                port: 
                  number: 80              
```

## Step-04: Deploy and Verify
```t
# Deploy Apps
kubectl apply -f kube-manifests/

# List Deployments
kubectl get deploy

# List Pods
kubectl get pods

# List Services
kubectl get svc

# List Ingress
kubectl get ingress
```

## Step-05: Access Applications
```t
# Access Application (curl)
curl http://<INGRESS-IP>/myapps/app1/index.html

# Access Application (Browser)
http://<INGRESS-IP>/myapps/app1/index.html
```

## Step-06: Verify Application Gateway Ingress Pod Logs
```t
# Verify ingress-appgw pod Logs 
kubectl -n kube-system logs -f $(kubectl -n kube-system get po | egrep -o 'ingress-appgw-deployment[A-Za-z0-9-]+')
```

## Step-07: Verify Azure AKS Cluster using Azure Portal
- **Workloads:** Review workloads
- **Services and Ingresses Tab:** Review services

## Step-08: Verify Application Gateway using Azure Portal
- **Backend Settings Tab:**   
  - Review `Override backend path`
- **Rules Tab:** Review Rules Tab 

## Step-09: Clean-Up Applications
```t
# Delete Apps
kubectl delete -f kube-manifests/

# Review Application Gateway Tabs
1. Backend Pools Tab 
2. Backend Settings Tab
3. Rules Tab
```
