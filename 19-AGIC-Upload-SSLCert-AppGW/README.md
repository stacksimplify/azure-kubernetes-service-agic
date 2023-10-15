# Azure AGIC Ingress - SSL Certs Upload to AppGW

## Step-01: Introduction
- We are going to understand and implement `appgw-ssl-certificate` Annotation
- We are going to upload an SSL Certificate to Azure AppGW and use it in our Ingress Manifest Annotation
```yaml
  annotations:
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    appgw.ingress.kubernetes.io/appgw-ssl-certificate: "app2sslcert"
```

## Step-02: App2 - Create Self-Signed SSL Certificates 
```t
# Change Directory 
cd SSL-SelfSigned-Certs

# Create your app2 private key:
openssl genrsa -out app2-ingress.key 2048

# Create your app2 certificate signing request:
openssl req -new -key app2-ingress.key -out app2-ingress.csr -subj "/CN=app2.kubeoncloud.com"

# Create your app2 certificate:
openssl x509 -req -days 7300 -in app2-ingress.csr -signkey app2-ingress.key -out app2-ingress.crt

# Convert Certificate and Key to PFX Format
openssl pkcs12 -export -in app2-ingress.crt -inkey app2-ingress.key -passout pass:certpass123 -out app2-ingress.pfx
```

## Step-03: Upload Certificate to Azure Application Gateway
```t
# Change Directory
cd SSL-SelfSigned-Certs

# Gather Information about AGIC
1. AGIC Name: agic-appgw
2. AGIC Resource Group: MC_agicdemo_agic-cluster_eastus

# List SSL Cets in AppGW
az network application-gateway ssl-cert list --resource-group MC_agicdemo_agic-cluster_eastus --gateway-name agic-appgw

# Upload Certificate to AppGW
az network application-gateway ssl-cert create \
  --resource-group MC_agicdemo_agic-cluster_eastus \
  --gateway-name agic-appgw \
  -n app2sslcert \
  --cert-file app2-ingress.pfx \
  --cert-password "certpass123"

# List SSL Cets in AppGW
az network application-gateway ssl-cert list --resource-group MC_agicdemo_agic-cluster_eastus --gateway-name agic-appgw

# Show app2sslcert from AppGW
az network application-gateway ssl-cert show -n app2sslcert --resource-group MC_agicdemo_agic-cluster_eastus --gateway-name agic-appgw

# Verify the same using Azure Portal
1. Go to Azure Portal -> App GW -> agic-appgw -> Settings -> Listeners
2. In "Listeners TLS Certificates" tab, we should find our "app2sslcert" listed
```

## Step-04: Review 03-Ingress-AppGW-SSL-Cert.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-selfsigned-ssl-tls
  annotations:
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
    appgw.ingress.kubernetes.io/appgw-ssl-certificate: "app2sslcert"
spec:
  ingressClassName: azure-application-gateway
  rules:
    - host: app2.kubeoncloud.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-nginx-clusterip-service
                port: 
                  number: 80 
```


## Step-05: Deploy Kubernetes Manifests and Verify
```t
# Change Directory
cd 19-AGIC-Upload-SSLCert-AppGW

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
kubectl describe ingress ingress-selfsigned-appgw-ssl

# Verify external-dns Controller logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
[or]
kubectl get pods
kubectl logs -f <External-DNS-Pod-Name>

# Verify Azure DNS
1. Go to DNS Zones -> kubeoncloud.com
2. Review the new DNS record created
```

## Step-06: Access Application
```t
# Access Application HTTP URL
http://app2.kubeoncloud.com   
Observation:
1. Should redirect to HTTPS URL

# Access Application HTTPS URL
https://app2.kubeoncloud.com

Observation:
1. You will get a warning "The certificate is not trusted because it is self-signed.". Click on "Accept the risk and continue"
2. Review SSL Certificate CN from browser
```


## Step-07: Clean-Up
```t
# Delete Application
kubectl delete -f kube-manifests/

# Delete SSL Certificate from AppGW (Optional)
1. Go to Azure Portal -> AppGW -> agic-appgw -> Settings -> Listeners 
2. In "Listeners TLS Certificates" Tab -> Open "app2sslcert" and click on "Delete"
3. Provide name of certificate to delete "app2sslcert"
4. Click on "Delete"
```

