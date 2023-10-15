# Azure Gateway Ingress - Rewrite Rule Set Annotation

## Step-01: Introduction
- We are going to understand and implement Rewrite Rule Set Annotation
```yaml
# Azure Ingress Rewrite Rule Set Annotation
  annotations:
    appgw.ingress.kubernetes.io/rewrite-rule-set: my-headers-rewrite-ruleset
spec:
```

## Step-02: Review k8s Application Manifests
- **kube-manifests:** 
- 01-EchoServer-Deployment.yaml
- 02-EchoServer-ClusterIP-Service.yaml
- 03-Ingress-rewrite-rule-set.yaml

## Step-03: Review Ingress Service Manifests
- [Ingress API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/)
- [AGIC Rewrite Rule Set Annotation](https://azure.github.io/application-gateway-kubernetes-ingress/annotations/#rewrite-rule-set)
- **Ingress Manifest:** 03-Ingress-rewrite-rule-set.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echoserver-ingress-service
  annotations:
    appgw.ingress.kubernetes.io/rewrite-rule-set: my-headers-rewrite-ruleset
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

## Step-04: Create Rewrite Rule Set in Application Gateway
- Go to AppGw -> agic-appgw -> Settings -> Rewrites
### Name and Association Tab
- **Name:** my-headers-rewrite-ruleset
- **Routing Rules | Paths:** Select the Rule
- Click **Next**
### Rewrite Rule Configuration
- Click on **Add rewrite rule**
- **Rewrite rule name:** custom-mydomain-header
- Click on **Click to configure this action**
- **Rewrite Type:** Request Header
- **Action Type:** Set
- **Header name:** Select `Custom Header`
- **Custom header:** mydomain
- **Header value:** stacksimplify.com
- Click **ok**
- Click on **Create**


## Step-05: Deploy and Verify
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
curl http://<INGRESS-IP>/

# Access Application (Browser)
http://<INGRESS-IP>/
Observation: 
1. You should see that rewrite rule executed and header `mydomain=stacksimplify.com` sent to backend application server
```

## Step-06: Clean-Up Applications
```t
# Delete Apps
kubectl delete -f kube-manifests/

# Review Application Gateway Tabs
1. Backend Pools Tab 
2. Backend Settings Tab
3. Rules Tab
```
