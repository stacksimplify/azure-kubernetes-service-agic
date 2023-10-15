# AGIC Ingress - SSL Redirect

## Step-01: Introduction
- We are going to understand and implement the AGIC Ingress SSL Redirect Annotation
```yaml
# AGIC Ingress SSL Redirect Annotation
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
```

## Step-02: Review 03-Ingress-SSL-TLS.yaml
- [AGIC SSL Redirect Annotation](https://azure.github.io/application-gateway-kubernetes-ingress/annotations/#ssl-redirect)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-selfsigned-ssl-tls
  annotations:
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
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

## Step-03: Deploy and Verify
```t
# Deploy updated Ingress Manifest
kubectl apply -f kube-manifests/

# Access Application (HTTP URL)
http://app1.kubeoncloud.com
Observation:
1. HTTP URL should redirect to HTTPS URL

# Review Azure Application Gateway Listeners
1. Go to Azure Application Gateway -> agic-appgw -> Settings -> Listeners
2. We should see a 80 listener created, open and review the associated Rule
3. In Rule, Backend Targets section we should 
Target Type: Redirection
Redirection Type: permanent
Target Listener: this will be 443 listener
```

## Step-04: Clean-Up
```t
# Delete Application
kubectl delete -f kube-manifests/

# Delete Kubernetes Secret
kubectl delete secret app1-secret
kubectl get secrets

# Delete SSL Certificate from AppGW
1. Go to Azure Portal -> AppGW -> agic-appgw -> Settings -> Listeners 
2. In "Listeners TLS Certificates" Tab -> Open "cert-default-app1-secret" and click on "Delete"
3. Provide name of certificate to delete "cert-default-app1-secret"
4. Click on "Delete"
```