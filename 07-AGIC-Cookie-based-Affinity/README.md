# Azure Gateway Ingress - Cookie Based Affinity Annotation

## Step-01: Introduction
- We are going to understand and implement a Cookie based Affinity Annotations listed below
```yaml
# Azure Ingress Cookie based Affinity Annotations
  annotations:
    appgw.ingress.kubernetes.io/cookie-based-affinity: "true"
    appgw.ingress.kubernetes.io/cookie-based-affinity-distinct-name: "true"        
```

## Step-02: Review k8s Application Manifests
- **kube-manifests Folder:** 01-kube-manifests-cookie-affinity
  - 01-EchoServer-Deployment.yaml
  - 02-EchoServer-ClusterIP-Service.yaml
  - 03-Ingress-cookie-affinity.yaml

## Step-03: Review Ingress Service Manifests
- [Ingress API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/)
- [Cookie Based Affinity Annotation](https://azure.github.io/application-gateway-kubernetes-ingress/annotations/#cookie-based-affinity)
- **Ingress Manifest:** 03-Ingress-cookie-affinity.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echoserver-ingress-service
  annotations:
    appgw.ingress.kubernetes.io/cookie-based-affinity: "true"
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
# Deploy Apps
kubectl apply -f 01-kube-manifests-cookie-affinity

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
Observation:
1. With curl requests distribute to different k8s pods

# Access Application (Browser)
http://<INGRESS-IP>/
Observation
1. With cookie affinity enabled, a AppGW cookie is set on browser and you will see that request always sent to same k8s pod. 
2. Review the cookie on browser using Google Chrome Developer Tools
MacOS Shortcut: Press CMD + OPTION + I
Windows Shortcut: Press F12 or Ctrl + Shift + I.
3. Cookie name looks as "ApplicationGatewayAffinity"
```

## Step-05: Verify Application Gateway using Azure Portal
- **Backend Settings Tab:**   
  - Review `Cookie Affinity`, it should be enabled

## Step-06: Delete 01-kube-manifests-cookie-affinity 
```t
# Uninstall k8s Resources
kubectl delete -f 01-kube-manifests-cookie-affinity/
```

## Step-07: Review k8s Application Manifests
- **kube-manifests Folder:** 02-kube-manifests-distinct-cookie
  - 01-EchoServer-Deployment.yaml
  - 02-EchoServer-ClusterIP-Service.yaml
  - 03-Ingress-cookie-affinity-distinct.yaml

## Step-08: Review Ingress Service Manifests
- [Ingress API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/)
- [Distinct Cookie Annotation](https://azure.github.io/application-gateway-kubernetes-ingress/annotations/#distinct-cookie-name)
- **Ingress Manifest:** 03-Ingress-cookie-affinity.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echoserver-ingress-service
  annotations:
    appgw.ingress.kubernetes.io/cookie-based-affinity: "true"
    appgw.ingress.kubernetes.io/cookie-based-affinity-distinct-name: "true"        
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

## Step-09: Deploy and Verify
```t
# Deploy Apps
kubectl apply -f 02-kube-manifests-distinct-cookie

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
Observation:
1. With curl requests distribute to different k8s pods

# Access Application (Browser)
http://<INGRESS-IP>/
Observation
1. With distinct cookie affinity enabled, a AppGW cookie per backend is set on browser and you will see that request always sent to same k8s pod. 
2. Review the cookie on browser using Google Chrome Developer Tools -> Network Tab
MacOS Shortcut: Press CMD + OPTION + I
Windows Shortcut: Press F12 or Ctrl + Shift + I.
3. Cookie name looks as below
appgw-affinity-<some-dynamic-id>
appgw-affinity-6b33b5c2a07e13ed106bb9a11903dffc	
```

## Step-10: Verify Application Gateway using Azure Portal
- **Backend Settings Tab:**   
  - Review `Cookie Affinity`, it should be enabled
  - `Affinity Cookie name` also contains value something like `appgw-affinity-6b33b5c2a07e13ed106bb9a11903dffc`. This should match the cookie name we have seen via Google Developer Tools Network Tab

## Step-11: Clean-Up Applications
```t
# Delete Apps
kubectl delete -f 02-kube-manifests-distinct-cookie/

# Review Application Gateway Tabs
1. Backend Pools Tab 
2. Backend Settings Tab
3. Rules Tab
```

