# Azure AGIC Ingress - TLS

## Step-00: Pre-requisites
1. Verify if Azure AKS Cluster is created
2. Verify if kubeconfig for kubectl is configured in your local terminal
```t
# Configure kubeconfig for kubectl
az aks get-credentials --resource-group agicdemo --name agic-cluster

# List Kubernetes Nodes
kubectl get nodes
```

3. ExternalDNS Controller should be installed and ready to use
```t
# List Deployments
kubectl get deploy

# List External DNS Pods
kubectl get pods
```

## Step-01: Introduction
1. Implement Self Signed SSL Certificates with Azure AGIC Ingress Service
2. Create SSL Certificates using OpenSSL.
3. Create Kubernetes Secret with SSL Certificate and Private Key
4. Reference these Kubernetes Secrets in Ingress Service **Ingress spec.tls**

## Step-02: App1 - Create Self-Signed SSL Certificates and Kubernetes Secrets
```t
# Change Directory 
cd SSL-SelfSigned-Certs

# Create your app1 private key:
openssl genrsa -out app1-ingress.key 2048

# Create your app1 certificate signing request:
openssl req -new -key app1-ingress.key -out app1-ingress.csr -subj "/CN=app1.kubeoncloud.com"

# Create your app1 certificate:
openssl x509 -req -days 7300 -in app1-ingress.csr -signkey app1-ingress.key -out app1-ingress.crt

# Create a Secret that holds your app1 certificate and key:
cd SSL-SelfSigned-Certs
kubectl create secret tls app1-secret  --cert app1-ingress.crt --key app1-ingress.key

# List Secrets
kubectl get secrets
```

## Step-03: Review App1 kube-manifests
1. 01-myapp-Deployment.yaml
2. 02-myapp-ClusterIP-Service.yaml

## Step-04: Review 03-Ingress-SSL-TLS.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-app1-ssl-tls
spec:
  ingressClassName: azure-application-gateway
  # SSL Certs - Associate using Kubernetes Secrets         
  tls:
  - secretName: app1-secret
  rules:
    - host: app1.kubeoncloud.com
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
# Deploy Kubernetes Manifests
kubectl apply -f kube-manifests

# List Deployments
kubectl get deploy

# List Pods
kubectl get pods

# List Services
kubectl get svc

# List Ingress Services
kubectl get ingress

# Describe Ingress Service
kubectl describe ingress ingress-selfsigned-ssl-tls

# Verify external-dns Controller logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
[or]
kubectl get pods
kubectl logs -f <External-DNS-Pod-Name>

# Verify Azure DNS
1. Go to DNS Zones -> kubeoncloud.com
2. Review the new DNS record created

# Verify SSL Certificate on Azure App Gateway
1. Go to Azure Application Gateway -> agic-appgw -> Settings -> Listeners
2. We should see a 443 listener created
3. Click on Tab "Listeners TLS Certificates"
4. We should see our SSL Certificate uploaded to Azure AppGw from Kubernetes secret we created "app1-secret"
5. Review the 443 Listener Port, Certificate, Listener Type, Host Type and Host names
```

## Step-06: Access Application
```t
# Access Application using HTTPS URL
https://app1.kubeoncloud.com

Observation:
1. You will get a warning "The certificate is not trusted because it is self-signed.". Click on "Accept the risk and continue"

# Access Application using HTTP
http://app1.kubeoncloud.com

Observation:
1. There is no port 80 listener so request should fail
2. We can create a SSL Redirect from port 80 to port 443 (HTTP to HTTPS) which we will do in our next demo
3. We will not clean-up our Application in this demo.
```

