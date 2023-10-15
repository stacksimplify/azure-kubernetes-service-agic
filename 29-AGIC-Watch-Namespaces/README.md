# AGIC - Watch Namespaces

## Step-01: Introduction
- How to direct AGIC to wathc specific namespaces by updating helm-config.yaml ?
```yaml
################################################################################
# Specify which kubernetes namespace the ingress controller will watch
# Default value is "default"
# Leaving this variable out or setting it to blank or empty string would
# result in Ingress Controller observing all acessible namespaces.
#
kubernetes:
    watchNamespace: production
```

## Step-02: Create Namespaces
```t
# Create Namespaces
kubectl create namespace staging
kubectl create namespace production
```

## Step-03: Review App1 and App2 Kubernetes Manifests
```t
# Folder: 01-app1-staging-kube-manifests
01-NginxApp1-Deployment.yaml
02-NginxApp1-ClusterIP-Service.yaml
03-Ingress-app1.yaml

# Folder: 02-app2-production-kube-manifests
01-NginxApp2-Deployment.yml
02-NginxApp2-ClusterIP-Service.yml
03-Ingress-app2.yaml
```

## Step-04: Deploy App1 and App2 Applications
```t
# Deploy App1 to Staging Namespace
kubectl apply -f 01-app1-staging-kube-manifests

# Deploy App2 to Production Namespace
kubectl apply -f 02-app2-production-kube-manifests

# Review App1 and App2 in Application Gateway
Backend Pools
Backend Settings
Rules
Listeners

# List Pods
kubect get pods -n staging
kubect get pods -n production

# List Ingress
kubectl get ingress -n staging
kubectl get ingress -n production

# Access App1 and App2
http://APPGW-IP/app1/index.html
http://APPGW-IP/app2/index.html

# Clean-Up
kubectl delete -f 01-app1-staging-kube-manifests
kubectl delete -f 02-app2-production-kube-manifests
```

## Step-05: Review helm-config-watch-prod-namespace.yaml
- **File:** helm-config-watch-prod-namespace.yaml
```yaml
# This file contains the essential configs for the ingress controller helm chart

# Verbosity level of the App Gateway Ingress Controller
verbosityLevel: 3

################################################################################
# Specify which application gateway the ingress controller will manage
#
appgw:
    subscriptionId: 82808767-144c-4c66-a320-b30791668b0a
    resourceGroup: agic-helm
    name: agic-appgw-helm
    usePrivateIP: false

    # Setting appgw.shared to "true" will create an AzureIngressProhibitedTarget CRD.
    # This prohibits AGIC from applying config for any host/path.
    # Use "kubectl get AzureIngressProhibitedTargets" to view and change this.
    shared: true

################################################################################
# Specify which kubernetes namespace the ingress controller will watch
# Default value is "default"
# Leaving this variable out or setting it to blank or empty string would
# result in Ingress Controller observing all acessible namespaces.
#
kubernetes:
    watchNamespace: production

################################################################################
# Specify the authentication with Azure Resource Manager
#
# Two authentication methods are available:
# - Option 1: AAD-Pod-Identity (https://github.com/Azure/aad-pod-identity)
armAuth:
    type: workloadIdentity
    identityClientID: ec9544f7-3a0a-492d-82ad-518ff4dfd80b

## Alternatively you can use Service Principal credentials
# armAuth:
#    type: servicePrincipal
#    secretJSON: <<Generate this value with: "az ad sp create-for-rbac --subscription <subscription-uuid> --sdk-auth | base64 -w0" >>

################################################################################
# Specify if the cluster is RBAC enabled or not
rbac:
    enabled: true # true/false
```


## Step-06: Upgrade Helm Release and Verify
```t
# List Helm Releases
helm list

# Add Helm Repository (if not added in local terminal)
helm repo add application-gateway-kubernetes-ingress https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/

# Upgrade Helm Release 
helm upgrade ingress-azure application-gateway-kubernetes-ingress/ingress-azure -f helm-config-watch-prod-namespace.yaml

# List Helm Releases
helm list 
helm list --output=yaml
Observation:
1. Helm release will be updated with new revision

# List Kubernetes Pods
kubectl get pods
```

## Step-07: Deploy App1 and App2 Applications
```t
# Deploy App1 to Staging Namespace
kubectl apply -f 01-app1-staging-kube-manifests

# Deploy App2 to Production Namespace
kubectl apply -f 02-app2-production-kube-manifests

# Review App1 and App2 in Application Gateway
Backend Pools
Backend Settings
Rules
Listeners
Observation:
1. Only App2 deployed to production namespace related entries will be present in Application Gateway

# List Pods
kubect get pods -n staging
kubect get pods -n production

# List Ingress
kubectl get ingress -n staging
kubectl get ingress -n production
Observation: 
1. Only App2 from production namespace will have the AppGw Public IP
2. For App1 from staging namespace will not have any IP associated

# Access App2 from Production Namespace
http://APPGW-IP/app2/index.html

# Clean-Up
kubectl delete -f 01-app1-staging-kube-manifests
kubectl delete -f 02-app2-production-kube-manifests
```

## Step-08: Clean-up
```t
# Helm Uninstall
helm list
helm uninstall ingress-azure

# Delete Resource Group
Resource Group: agic-helm
1. Verify AKS Cluster got deleted
2. Veify AppGw got deleted
```