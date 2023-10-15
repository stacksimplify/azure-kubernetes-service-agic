# AGIC Ingress - SSL using Lets Encrypt

## Step-01: Introduction
- Implement SSL using Lets Encrypt

## Step-02: Install Cert Manager using Helm

### Step-02-01: Install Helm CLI
```t
# Install Helm CLI
https://helm.sh/docs/intro/install/

# HELM MASTERCLASS
https://github.com/stacksimplify/helm-masterclass
## For Windows install by downloading Package
https://github.com/stacksimplify/helm-masterclass/tree/main/01-Install-Docker-Desktop-and-HelmCLI#step-08-windows-install-helm-cli-using-package

# Verify Helm Version
helm version
```

### Step-02-02: Create cert-manager Namespace
```t
# Create Namespace 
kubectl create namespace cert-manager

# Label the  cert-manager namespace to disable resource validation
kubectl label namespace cert-manager cert-manager.io/disable-validation=true
```

### Step-02-03: Install Cert Manager using Helm
```t
# Review Releases and Update latest release number below
https://github.com/cert-manager/cert-manager/releases/ 

# To Install CRDs manually without HELM
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.crds.yaml

# To Uninstall CRDs manually
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.crds.yaml

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# List Helm Repos
helm repo list

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.13.0 \
  --set installCRDs=true

# List Helm Releases in cert-manager namespace
helm list -n cert-manager

# helm status in cert-manager namespace
helm status cert-manager --show-resources -n cert-manager
Obserevation: 
1. Review all the resources created as part of this Helm Release

# Verify Cert Manager pods in cert-manager namespace
kubectl get pods --namespace cert-manager

# Verify Cert Manager Services
kubectl get svc --namespace cert-manager

# Verify All Kubernetes Resources created in cert-manager namespace
kubectl get all --namespace cert-manager
```

## Step-03: Review or Create Cluster Issuer Kubernetes Manifest
### Step-03-01: Review Cluster Issuer Kubernetes Manifest
- Create or Review Cert Manager Cluster Issuer Kubernetes Manifest
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: dkalyanreddy@gmail.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt
    solvers:
      - http01:
          ingress:
            class: azure/application-gateway
```

### Step-03-02: Deploy Cluster Issuer
```t
# Deploy Cluster Issuer
kubectl apply -f 01-CertManager-ClusterIssuer/cluster-issuer.yml

# List Cluster Issuer
kubectl get clusterissuer

# Describe Cluster Issuer
kubectl describe clusterissuer letsencrypt
```


## Step-04: Review Application NginxApp1,2 K8S Manifests
- 01-NginxApp1-Deployment.yaml
- 02-NginxApp1-ClusterIP-Service.yaml
- 03-NginxApp2-Deployment.yaml
- 04-NginxApp2-ClusterIP-Service.yaml

## Step-05: Create or Review Ingress SSL Kubernetes Manifest
- **File Name:** 05-Ingress-SSL-LetsEncrypt.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-ssl-letsencrypt
  annotations:
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt    
spec:
  ingressClassName: azure-application-gateway
  tls:
  - secretName: sapp1-kubeoncloud-secret 
    hosts:
    - sapp1.kubeoncloud.com    
  - secretName: sapp2-kubeoncloud-secret                     
    hosts:
    - sapp2.kubeoncloud.com  
  rules:
    - host: sapp1.kubeoncloud.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app1-nginx-clusterip-service
                port: 
                  number: 80
    - host: sapp2.kubeoncloud.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app2-nginx-clusterip-service
                port: 
                  number: 80                         
```

## Step-06 Deploy Kubernetes Manifests & Verify
- Certificate Request, Generation, Approval and Download and be ready might take from 1 hour to couple of days, if we make any mistakes it might also fail.
- For me it took, only 5 minutes to get the certificate from **https://letsencrypt.org/**
```t
# Verify if Cluster Issuer is already deployed
kubectl get clusterissuer

# Deploy Application
kubectl apply -f 02-kube-manifests/

# Verify Pods
kubectl get pods

# Verify Kubernetes Secrets
kubectl get secrets

# YAML Output of Kubernetes Secrets
kubectl get secret sapp1-kubeoncloud-secret -o yaml
kubectl get secret sapp2-kubeoncloud-secret -o yaml
Observation:
1. Review tls.crt and tls.key

# Verify SSL Certificates (It should turn to True)
kubectl get certificate

## Sample Output
NAME                       READY   SECRET                     AGE
sapp1-kubeoncloud-secret   True    sapp1-kubeoncloud-secret   2m48s
sapp2-kubeoncloud-secret   True    sapp2-kubeoncloud-secret   2m48s

# Verify Cert Manager Pod Logs
kubectl -n cert-manager get pods 
kubectl -n cert-manager logs -f <cert-manager-POD-NAME>
kubectl -n cert-manager logs -f cert-manager-5c84984c7b-ft5hn


# Verify external-dns Controller logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
[or]
kubectl get pods
kubectl logs -f <External-DNS-Pod-Name>

# If NO External DNS Installed What we need to do ?
1. We need to manually add the DNS name in Azure DNS Zones before deploying "kube-manifests"
2. Get the Public IP from Azure Application Gateway -> agic-appgw -> Overview Tab
3. Go to DNS Zones -> Add Record set 
sapp1.kubeoncloud.com
sapp2.kubeoncloud.com
```


## Step-07: Verify Settings in Azure Portal  for Azure Application Gateway
```t
# 1. AppGw Backend Pools
1. We should see App1 and App2 two backend pools created

# 2. AppGw Backend Settings
1. We should see App1 and App2 two backend settings created


# 3. AppGw Listeners
1. We should see two port 80 listeners created to redirect to port 443 for App1 and App2
2. We should see two port 443 listeners created to serve content for App1 and App2 with hostnames as sapp1.kubeoncloud.com, sapp2.kubeoncloud.com

# 4. AppGw Listener TLS Certificates 
1. We should see below listed two LetsEncrypt auto-uploaded to Application Gateway when we deployed our Ingress Manifest
1.1 cert-default-sapp1-kubeoncloud-secret
1.2 cert-default-sapp2-kubeoncloud-secret
2. This happened because we used "spec.tls.secretName" in our Ingress Service

# 5. AppGw Rules
1. Two Port 80 Rules: In Backend Targets
1.1 Target Type: Redirection
1.2 Redirection type: permanent
1.3 Redirection target: Listener
1.4 Target listener: Port 443 listener

2. Two Port 443 Rules created


# 6. AppGw Health Probes
1. Two HTTP Health probes created for App1 and App2
```


## Step-08: Access Application
```t
# Application HTTP URLs
http://sapp1.kubeoncloud.com/app1/index.html
http://sapp2.kubeoncloud.com/app2/index.html
Observation:
1. Should redirect to HTTPS url
2. We have added AGIC ssl-redirect annotation in Ingress Manifest

# Application HTTPS URLs
https://sapp1.kubeoncloud.com/app1/index.html
https://sapp2.kubeoncloud.com/app2/index.html
Observation
1. Review SSL Certificate from browser after accessing URL
2. We should valid SSL certificate generated by LetsEncrypt

```
## Step-09: Clean-Up
```t
# Delete Applications
kubectl delete -f 02-kube-manifests/
```

## Cert Manager References
- https://cert-manager.io/docs/installation/#default-static-install
- https://cert-manager.io/docs/installation/helm/
- https://docs.cert-manager.io/
- https://cert-manager.io/docs/installation/helm/#1-add-the-jetstack-helm-repository
- https://cert-manager.io/docs/configuration/
- https://cert-manager.io/docs/tutorials/acme/nginx-ingress/#step-6---configure-a-lets-encrypt-issuer
- https://letsencrypt.org/how-it-works/

  