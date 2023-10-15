# Azure AGIC Ingress - Health Probes (AGIC Annotations)

## Step-01: Introduction
- We are going to understand and implement Health Probes using AGIC Annotations
- [AGIC Health Probe Annotations](https://azure.github.io/application-gateway-kubernetes-ingress/annotations/#health-probe-hostname)
```yaml
  annotations:
    # Health Probe Annotations
    appgw.ingress.kubernetes.io/health-probe-hostname: "myapp1.stacksimplify.com"
    appgw.ingress.kubernetes.io/health-probe-port: "80"
    appgw.ingress.kubernetes.io/health-probe-path: /app1/index.html
    appgw.ingress.kubernetes.io/health-probe-status-codes: "200-205, 206"
    appgw.ingress.kubernetes.io/health-probe-interval: "32"    
    appgw.ingress.kubernetes.io/health-probe-timeout: "32"
    appgw.ingress.kubernetes.io/health-probe-unhealthy-threshold: "4"
```

## Step-02: Health Probe Annotations
- [AGIC Health Probe Annotations](https://azure.github.io/application-gateway-kubernetes-ingress/annotations/#health-probe-path)
```t
# Review the below kube-manifests
# kube-manifests: 01-kube-manifests-readiness
- 01-NginxApp1-Deployment.yaml
- 02-NginxApp1-ClusterIP-Service.yaml
- 03-Ingress-Basic.yml

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
curl http://<INGRESS-IP>/app1/index.html

# Access Application (Browser)
http://<INGRESS-IP>/app1/index.html

# Verify Application Gateway Health Probe
- Go to AppGw -> agic-appgw -> Settings -> Health Probes
- Perform TEST

# Uninstall Apps
kubectl delete -f kube-manifests/
```
