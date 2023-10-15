# Ingress - Domain Name Based Routing

## Step-01: Introduction
- We are going to implement Domain Name based routing using Ingress
- We are going to use 3 applications for this.

## Step-02: Review k8s Application Manifests
- App1 Manifests
- App2 Manifests
- App3 Manifests

## Step-03: Review Ingress Service Manifests
- 01-Ingress-DomainName-Based-Routing-app1-2-3.yml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-app1-app2-app3
spec:
  ingressClassName: azure-application-gateway
  rules:
    - host: eapp1.kubeoncloud.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app1-nginx-clusterip-service
                port: 
                  number: 80
    - host: eapp2.kubeoncloud.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app2-nginx-clusterip-service
                port: 
                  number: 80
    - host: eapp3.kubeoncloud.com
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

## Step-04: Deploy and Verify
```t
# Deploy Apps
kubectl apply -R -f kube-manifests/

# List Pods
kubectl get pods

# List Services
kubectl get svc

# List Ingress
kubectl get ingress

# Verify External DNS pod to ensure record set got deleted
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')


# Verify Record set got automatically deleted in DNS Zones
# Template Command
az network dns record-set a list -g <Resource-Group-dnz-zones> -z <yourdomain.com>

# Replace DNS Zones Resource Group and yourdomain
az network dns record-set a list -g dns-zones -z kubeoncloud.com

# Additionally you can review via Azure Portal
Go to Portal -> DNS Zones -> <YOUR-DOMAIN>
Review records in "Overview" Tab
```

## Step-05: Review Azure Application Gateway using Azure Portal
```t
# Review Azure AppGW Listeners
1. Go to Azure AppGW -> agic-appgw -> Settings -> Listeners
2. We should see 3 Listeners created 1 for each Application 
3. All 3 are Port 80 listeners
4. Can we create 3 listeners with same port 80 ?
5. Yes we can create multiple listeners with same port provided there hostname is different
6. Open any one listener and review the "Host type" and "Host names" settings
```
## Step-06: Access Applications
```t
# Access App1
http://eapp1.kubeoncloud.com/app1/index.html

# Access App2
http://eapp2.kubeoncloud.com/app2/index.html

# Access MyApp
http://eapp3.kubeoncloud.com

```

## Step-07: Clean-Up Applications
```t
# Delete Apps
kubectl delete -R -f kube-manifests/

# Verify Record set got automatically deleted in DNS Zones
# Template Command
az network dns record-set a list -g <Resource-Group-dnz-zones> -z <yourdomain.com>

# Replace DNS Zones Resource Group and yourdomain
az network dns record-set a list -g dns-zones -z kubeoncloud.com

# Additionally you can review via Azure Portal
Go to Portal -> DNS Zones -> <YOUR-DOMAIN>
Review records in "Overview" Tab
```

