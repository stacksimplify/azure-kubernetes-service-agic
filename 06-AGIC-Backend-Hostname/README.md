# Azure Gateway Ingress - Backend Annotations Hostname

## Step-01: Introduction
- We are going to understand and implement a Backend Annotation listed below
```yaml
# Azure Ingress Backend Annotation
  annotations:
    appgw.ingress.kubernetes.io/backend-hostname: "internal.stacksimplify.com"   
```

## Step-02: Review k8s Application Manifests
- **kube-manifests:** 
- 01-EchoServer-Deployment.yaml
- 02-EchoServer-ClusterIP-Service.yaml
- 03-Ingress-backend-hostname.yaml

## Step-03: Review Ingress Service Manifests
- [Ingress API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/)
- [AGIC Backend Hostname Annotation](https://azure.github.io/application-gateway-kubernetes-ingress/annotations/#backend-hostname)
- **Ingress Manifest:** 03-Ingress-backend-hostname.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echoserver-ingress-service
  annotations:
    #appgw.ingress.kubernetes.io/backend-hostname: "internal.example.com"
spec:
  ingressClassName: azure-application-gateway
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: echoserver-clusterip-service
                port: 
                  number: 80          
```

## Step-04: Deploy and Verify
```t
# Deploy without Backend Hostname Annotation
1. First we will deploy without backend hostname annotation, review the output 
2. Then we will enable the backend hostname annotation and review the output

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
curl http://<INGRESS-IP>/

# Access Application (Browser)
http://<INGRESS-IP>/
```

## Step-05: Verify Application Gateway using Azure Portal
- **Backend Settings Tab:**   
  - Review `Override with new host name`

## Step-06: Uncomment Backend Hostname Annotation, Deploy and Verify
```t
# In Ingress manifest uncomment
  annotations:
    appgw.ingress.kubernetes.io/backend-hostname: "internal.stacksimplify.com"

# Deploy Apps
kubectl apply -f kube-manifests/

# List Ingress
kubectl get ingress

# Access Application (curl)
curl http://<INGRESS-IP>/

# Access Application (Browser)
http://<INGRESS-IP>/
```

## Step-07: Verify Application Gateway using Azure Portal
- **Backend Settings Tab:**   
  - Review `Override with new host name`

## Step-08: Clean-Up Applications
```t
# Delete Apps
kubectl delete -f kube-manifests/

# Review Application Gateway Tabs
1. Backend Pools Tab 
2. Backend Settings Tab
3. Rules Tab
```
