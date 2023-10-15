# Azure Gateway Ingress - Override Frontend Port

## Step-01: Introduction
- We are going to understand and implement how to Override frontend port using Ingress Service
```yaml
# Azure Ingress Override Frontend Port
  annotations:
    appgw.ingress.kubernetes.io/override-frontend-port: "8080"
```

## Step-02: Review k8s Application Manifests
- **kube-manifests:** 
- 01-myapp-Deployment.yaml
- 02-myapp-ClusterIP-Service.yaml
- 03-Ingress-override-frontend.yaml

## Step-03: Review Ingress Service Manifests
- [Ingress API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/)
- [AGIC Override Frontend Port Annotation](https://azure.github.io/application-gateway-kubernetes-ingress/annotations/#override-frontend-port)
- **Ingress Manifest:** 03-Ingress-override-frontend.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress-service
  annotations:
    appgw.ingress.kubernetes.io/override-frontend-port: "8080"
spec:
  ingressClassName: azure-application-gateway
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-nginx-clusterip-service
                port: 
                  number: 80      
```

## Step-03: Deploy and Verify
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

# Access Application (curl)
curl http://<INGRESS-IP>:8080/

# Access Application (Browser)
http://<INGRESS-IP>:8080/
```

## Step-04: Verify Application Gateway using Azure Portal
- **Listeners Tab:** Review Rules Tab 
  - You should find the port as `8080`

## Step-05: Clean-Up Applications
```t
# Delete Apps
kubectl delete -f kube-manifests/

# Review Application Gateway Tabs
1. Backend Pools Tab 
2. Backend Settings Tab
3. Rules Tab
```
