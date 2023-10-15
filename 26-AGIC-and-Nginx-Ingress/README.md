# AGIC Ingress - Multiple Ingress Controllers

## Step-01: Introduction
### NGINX Ingress Controller: What are we going to learn?
- We are going to create a **Static Public IP** for Ingress in Azure AKS
- Associate that Public IP to **Ingress Controller** during installation.
- We are going to create a namespace `ingress-nginx` for Ingress Controller where all ingress controller related things will be placed. 
- Create / Review Ingress Manifest
- We are going to understand about `Ingress Class` and `Ingress Class Name`
- We are going to deploy Applications to both Azure AGIC and Nginx Ingress with  `ingressClassName` and Verify
```yaml
# With ingressClassName in Ingress Manifest
## This will associate to Nginx Ingress Controller
spec:
  ingressClassName: nginx

## This will associate to Azure AGIC Controller
spec:
  ingressClassName: azure-application-gateway
```

## Step-02: Create Static Public IP
```t
# Get the resource group name of the AKS cluster 
az aks show --resource-group agicdemo --name agic-cluster --query nodeResourceGroup -o tsv

# TEMPLATE - Create a public IP address with the static allocation
az network public-ip create --resource-group <REPLACE-OUTPUT-RG-FROM-PREVIOUS-COMMAND> --name myAKSPublicIPForIngress --sku Standard --allocation-method static --query publicIp.ipAddress -o tsv

# REPLACE - Create Public IP: Replace Resource Group value
az network public-ip create --resource-group MC_agicdemo_agic-cluster_eastus --name myAKSPublicIPForNginxIngress --sku Standard --allocation-method static --query publicIp.ipAddress -o tsv
```
- Make a note of Static IP which we will use in next step when installing Ingress Controller
```t
# Make a note of Public IP created for Ingress
20.121.21.149
```

## Step-03: Install Nginx Ingress Controller
- [ingress-nginx Controller Git Repo](https://github.com/kubernetes/ingress-nginx)
```t
# List Ingress Class
kubectl get ingressclass

# Install Helm3 (if not installed)
brew install helm

# Create a namespace for your ingress resources
kubectl create namespace ingress-nginx

# Add the official stable repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Search Helm Repo
helm search repo ingress-nginx
helm search repo ingress-nginx --versions

#  Customizing the Chart Before Installing. 
helm show values ingress-nginx/ingress-nginx

# Use Helm to deploy an NGINX ingress controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --set controller.replicaCount=2 \
    --set controller.service.externalTrafficPolicy=Local \
    --set controller.service.loadBalancerIP="REPLACE_STATIC_IP" 

# Replace Static IP captured in Step-02 (without beta for NodeSelectors)
helm install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-nginx \
    --set controller.replicaCount=2 \
    --set controller.service.externalTrafficPolicy=Local \
    --set controller.service.loadBalancerIP="20.121.21.149"     

# List Ingress Class
kubectl get ingressclass

# List Services with labels
kubectl get service -l app.kubernetes.io/name=ingress-nginx --namespace ingress-nginx
kubectl get service -n ingress-nginx

# List Pods
kubectl get pods -n ingress-nginx
kubectl get all -n ingress-nginx

# Helm List
helm list -n ingress-nginx

# Helm status --show-resources
helm status --show-resources <RELEASE-NAME> -n ingress-nginx
helm status --show-resources ingress-nginx -n ingress-nginx
Observation:
1. Will show all the kubernetes resources created as part of this Helm Chart in Helm Release


# Access Public IP
http://<Public-IP-created-for-Ingress>
http://20.121.21.149

# Output should be
404 Not Found from Nginx

# Verify Load Balancer on Azure Mgmt Console
Primarily refer Settings -> Frontend IP Configuration
```

## Step-04: Review Application k8s manifests with Ingress Class
### Step-04-01: kube-manifests folder: kube-manifests
#### Folder: 01-App1-NginxIC**
- 01-NginxApp1-Deployment.yml
- 02-NginxApp1-ClusterIP-Service.yml
- 03-Ingress.yml
```yaml
spec:
  ingressClassName: nginx 
```
#### Folder: 02-App2-AGIC
- 01-NginxApp2-Deployment.yml
- 02-NginxApp2-ClusterIP-Service.yml
- 03-Ingress.yml
```yaml
spec:
  ingressClassName: azure-application-gateway    
```

### Step-04-02: Deploy Application k8s manifests and verify
```t
# Deploy
kubectl apply -R -f kube-manifests

# List Pods
kubectl get pods

# List Services
kubectl get svc

# List Ingress
kubectl get ingress

# Access Applications
http://<NGINXIC-IP>/app1/index.html
http://<AGIC-IP>/app2/index.html

# Verify Load Balancers
1. Load Balancer
2. Application Gateway
```

### Step-04-03: Clean-Up Apps
```t
# Delete Apps
kubectl delete -R -f kube-manifests
```



## Ingress Annotation Reference
- https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations/

## Other References
- https://github.com/kubernetes/ingress-nginx
- https://github.com/kubernetes/ingress-nginx/blob/master/charts/ingress-nginx/values.yaml
- https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.34.1/deploy/static/provider/cloud/deploy.yaml
- https://kubernetes.github.io/ingress-nginx/deploy/#azure
- https://helm.sh/docs/intro/install/
- https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#ingress-v1-networking-k8s-io
- [Kubernetes Ingress API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/#ingress-v1-networking-k8s-io)
- [Ingress Path Types](https://kubernetes.io/docs/concepts/services-networking/ingress/#path-types)

## Important Note
```
Ingress Admission Webhooks
With nginx-ingress-controller version 0.25+, the nginx ingress controller pod exposes an endpoint that will integrate with the validatingwebhookconfiguration Kubernetes feature to prevent bad ingress from being added to the cluster. This feature is enabled by default since 0.31.0.
```
