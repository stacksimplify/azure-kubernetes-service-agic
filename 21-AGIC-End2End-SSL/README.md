# Azure AGIC Ingress - End to End SSL

## Step-01: Introduction
- We are going to implement End to End SSL Usecase as part of this demo
- We are going to use following AGIC Annotations for the same
```yaml
  annotations:
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    # Backend Annotations
    appgw.ingress.kubernetes.io/backend-hostname: "backend.kubeoncloud.com" # Optional
    appgw.ingress.kubernetes.io/backend-protocol: "https"
    appgw.ingress.kubernetes.io/appgw-trusted-root-certificate: "backend-tls"
```

## Step-02: Review SSL-SelfSigned-Certs folder
- Folder `SSL-SelfSigned-Certs` is a copy from previous demo `20-AGIC-Backend-SSL-Application` which contains the files 
  - backend-ssl.crt
  - backend-ssl.csr
  - backend-ssl.key
- We will need `backend-ssl.crt` in our next step

## Step-03: Upload Backend Trusted Certificate to Azure Application Gateway
- If we have a Trusted Root CA Certificate, we are going to upload that as part of this step.
- If we don't have a Trusted Root CA Certificate, if we have a self-signed certificate, then we will upload that self-signed certificate only
- In our case, we are using self-signed certificate, so we will upload `backend-ssl.crt` to Azure Application Gateway
```t
# Change Directory
cd SSL-SelfSigned-Certs

# Gather Information about AGIC
1. AGIC Name: agic-appgw
2. AGIC Resource Group: MC_agicdemo_agic-cluster_eastus

# Upload the backend certificate's root certificate to Application Gateway
# If no ROOT CA is available and if it is self-signed, upload backend certificate itself
az network application-gateway root-cert create \
    --gateway-name agic-appgw  \
    --resource-group MC_agicdemo_agic-cluster_eastus \
    --name backend-tls \
    --cert-file backend-ssl.crt

# List AppGW Trusted Root Certificates    
az network application-gateway root-cert list --gateway-name agic-appgw  --resource-group MC_agicdemo_agic-cluster_eastus 

# Show AppGw Trusted Root Certificates
az network application-gateway root-cert show --gateway-name agic-appgw  --resource-group MC_agicdemo_agic-cluster_eastus --name backend-tls
```

## Step-04: How to verify if our Trusted Root Certificate is uploaded successfully using Azure Portal ?
```t
# Verify Root Cert (backend cert) uploaded using Azure Portal
1. Go to Azure App Gatewat -> agic-appgw -> Settings -> Backend Settings
2. Click on "ADD"
3. Backend Protocol: HTTPS
4. Backend server’s certificate is issued by a well-known CA: No
5. Upload Root CA certificate: Select Existing
6. Choose Certificate: backend-tls (We should find our uploaded cert here)
7. Click on "Cancel"
```

## Step-05: Create Frontend Self-signed SSL Certificate 
```t
# Change Directory 
cd SSL-SelfSigned-Certs

# Create your frontend private key:
openssl genrsa -out frontend-ingress.key 2048

# Create your frontend certificate signing request:
openssl req -new -key frontend-ingress.key -out frontend-ingress.csr -subj "/CN=frontend.kubeoncloud.com"

# Create your frontend certificate:
openssl x509 -req -days 7300 -in frontend-ingress.csr -signkey frontend-ingress.key -out frontend-ingress.crt
```

## Step-06: Create Frontend TLS Kubernetes Secret
```t
# Change Directory
cd SSL-SelfSigned-Certs

# Create a Secret that holds your app1 certificate and key:
kubectl create secret tls frontend-tls-secret  --cert frontend-ingress.crt --key frontend-ingress.key

# List Secrets
kubectl get secrets
```

## Step-07: Review Kubernetes Manifests
- 01-myapp-configmap.yaml
- 02-myapp-Deployment.yaml
- 03-myapp-ClusterIP-Service.yaml
```t
# Change in 03-myapp-ClusterIP-Service.yaml
  type: LoadBalancer

# To
  type: ClusterIP
```

## Step-08: Review 04-Ingress-e2e-ssl.yaml
- [Kubernetes API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/)
- Review `tls.secretName` and `tls.hosts` in Ingress Spec from Kubernetes API Reference
- [AGIC appgw-trusted-root-certificate Annotation](https://azure.github.io/application-gateway-kubernetes-ingress/annotations/#appgw-trusted-root-certificate)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-e2e-ssl
  annotations:
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    # Backend Annotations
    appgw.ingress.kubernetes.io/backend-hostname: "backend.kubeoncloud.com" # Optional
    appgw.ingress.kubernetes.io/backend-protocol: "https"
    appgw.ingress.kubernetes.io/appgw-trusted-root-certificate: "backend-tls"
spec:
  ingressClassName: azure-application-gateway
  # SSL Certs - Associate using Kubernetes Secrets         
  tls:
  - secretName: frontend-tls-secret
    hosts:
    - frontend.kubeoncloud.com
  rules:
    - host: frontend.kubeoncloud.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-nginx-clusterip-service
                port: 
                  number: 443
```

## Step-09: Deploy Kubernetes Manifests and Verify
```t
# Change Directory
cd 21-AGIC-End2End-SSL

# Deploy Kubernetes Manifests
kubectl apply -f kube-manifests/

# List Deployments
kubectl get deploy

# List Pods
kubectl get pods

# List Services
kubectl get svc

# List Ingress Services
kubectl get ingress

# Describe Ingress Service
kubectl describe ingress ingress-e2e-ssl

# Verify external-dns Controller logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
[or]
kubectl get pods
kubectl logs -f <External-DNS-Pod-Name>

# If NO External DNS Installed What we need to do ?
1. We need to manually add the DNS name in Azure DNS Zones before deploying "kube-manifests"
2. Get the Public IP from Azure Application Gateway -> agic-appgw -> Overview Tab
3. Go to DNS Zones -> Add Record set 

# Verify Azure DNS
1. Go to DNS Zones -> kubeoncloud.com
2. Review the new DNS record created
```

## Step-10: Verify Settings in Azure Portal  for Azure Application Gateway
```t
# 1. AppGw Backend Pools
1. Review Backend pool

# 2. AppGw Backend Settings
1. Backend port: 443
2. Backend protocol: Https
3. Backend server’s certificate is issued by a well-known CA: No
4. Upload Root CA certificate: backend-tls
5. Override with new host name: backend.kubeoncloud.com
6. Custom probe


# 3. AppGw Listeners
1. 80 Listener 
1.1 Associated a rule -> which is a ssl-redirect

2. 443 Listener 
2.1 Port: 443
2.2 Choose a certificate: Select existing
2.3 Certificate: cert-default-frontend-tls-secret
2.4 Host type: Multiple/Wildcard
2.5 Host names: frontend.kubeoncloud.com

# 4. AppGw Listener TLS Certificates 
1. frontend-tls-secret uploaded to AppGw
2. We should see that with name "cert-default-frontend-tls-secret"
3. This happened because we used "spec.tls.secretName" in our Ingress Service

# 5. AppGw Rules
1. Port 80 Rule: In Backend Targets
1.1 Target Type: Redirection
1.2 Redirection type: permanent
1.3 Redirection target: Listener
1.4 Target listener: Port 443 listener

2. Port 443 Rule: In Backend Targets
2.1 Target Type: Backend Pool
2.2 Backend Target: 443 backend pool
2.3 Backend Settings: 443 backend seetings

# 6. AppGw Health Probes
1. Protocol: HTTPS
2. Host: backend.kubeoncloud.com
3. Path: /
4. Backend Settings: 443 backend settings
```

## Step-11: Access Application
```t
# Access Application
http://frontend.kubeoncloud.com
Observation:
1. should redirect to HTTPS url
2. Review SSL Certificate whose CN should frontend.kubeoncloud.com which we used during creation of frontend SSL Certificate
3. In parallel, also review container logs

# Access Kubernetes pod logs
kubectl get pods
kubectl logs -f <POD-NAME>
kubectl logs -f myapp-nginx-deployment-65b8578fcc-dwlz4

# Additional Verification by running tcpdump in pod
## Connect to pod
kubectl exec --stdin --tty <POD-NAME> -- /bin/bash
kubectl exec --stdin --tty  myapp-nginx-deployment-65b8578fcc-dwlz4 -- /bin/bash

## Commands inside pod
hostname
apt-get update
apt-get install tcpdump
tcpdump -l
Access URL in browser: http://frontend.kubeoncloud.com
exit
```

## Step-13: Clean-Up
```t
# Delete Applications
kubectl delete -f kube-manifests/
```