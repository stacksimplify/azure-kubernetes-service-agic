# AGIC Ingress - Create WAF Policy and Associate using Annotation

## Step-01: Introduction
- Create a WAF Policy
- Configure WAF Policy in our Ingress Service using below listed Annotation
- Understand how WAF Policy is associated to Application Gateway when "Route Paths (/app1, /app2) are used and when Root Context (/) is used
  - When `Route Paths` used WAF Policy associates to `Route Paths`
  - When `Root Context` used WAF Policy associates to `Listeners`
  - We have two demos in this section to understand the above concept in detail
```yaml
metadata:
  name: ingress-waf
  annotations:
    appgw.ingress.kubernetes.io/waf-policy-for-path: "/subscriptions/82808767-144c-4c66-a320-b30791668b0a/resourceGroups/agicdemo/providers/Microsoft.Network/ApplicationGatewayWebApplicationFirewallPolicies/myapp1-waf-policy1"
```

## Step-02: Create Web Application Firewall Policy
```t
# Create Web Application Firewall Policy
1. Go to Portal -> WAF -> Create

# 2. Basics Tab
2.1 Policy for: Regional WAF (Application Gateway)
2.2 Subscription: MY-SUBSCRIPTION
2.3 Resource Group: agicdemo
2.4 Policy Name: myapp1-waf-policy1
2.5 Location: East US
2.6 Policy State: Checked
2.7 Policy Mode: Prevention

# 3. Managed Rules Tab
Leave to defaults

# 4. Policy Settings Tab
Leagve to defaults

# 5. Custom Rules Tab;
1. Click on "Add custom rule"
2. custom rule name: BlockMyIPAddress
3. Enable Rule: checked
4. Rule type: Match
5. Priority: 5
6. Conditions
6.1 Match Type: IP Address
6.2 IP Address or Range: UPDATE-PUBLIC-IP HERE
Get your Public IP using URL: https://nordvpn.com/what-is-my-ip/
6.3 THEN DENY TRAFFIC
7. Click on Add

# 6. Associations Tab
1. Leave to defaults
2. Discuss about Application Gateway, Listener, Route Path

# 7. Tags Tab
1. Leave to defaults

# 8. Review + Create
1. Review and create

# After creation make a note of Resource ID
1. Go to Portal -> WAF -> myapp1-waf-policy1 -> Settings -> Properties
2. Copy Resource ID from Properties section
3. We are going to use this Resoruce ID in Ingress Service as value to Annotation "appgw.ingress.kubernetes.io/waf-policy-for-path:"
Resource ID: 
/subscriptions/82808767-144c-4c66-a320-b30791668b0a/resourceGroups/agicdemo/providers/Microsoft.Network/applicationGatewayWebApplicationFirewallPolicies/myapp1-waf-policy1
```

## Step-03: Review Kubernetes Manifests
- **kube-manifests folder:** 01-kube-manifests-scope-RoutePath
- 01-NginxApp1-Deployment.yaml
- 02-NginxApp1-ClusterIP-Service.yaml

## Step-04: Review Ingress Manifest 03-Ingress-WAF-Policy.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-waf
  annotations:
    appgw.ingress.kubernetes.io/waf-policy-for-path: "/subscriptions/82808767-144c-4c66-a320-b30791668b0a/resourceGroups/agicdemo/providers/Microsoft.Network/ApplicationGatewayWebApplicationFirewallPolicies/myapp1-waf-policy1"
spec:
  ingressClassName: azure-application-gateway
  rules:
    - host: myapp1.kubeoncloud.com
      http:
        paths:
          - path: /app1
            pathType: Prefix
            backend:
              service:
                name: app1-nginx-clusterip-service
                port: 
                  number: 80
```


## Step-05: Deploy Kubernetes Manifests and Verify
```t
# Change Directory
cd 24-AGIC-WAF-Policy

# Deploy Kubernetes Manifests
kubectl apply -f 01-kube-manifests-scope-RoutePath/

# List Deployments
kubectl get deploy

# List Pods
kubectl get pods

# List Services
kubectl get svc

# List Ingress Services
kubectl get ingress

# Describe Ingress Service
kubectl describe ingress ingress-waf

# Verify external-dns Controller logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
[or]
kubectl get pods
kubectl logs -f <External-DNS-Pod-Name>

# If NO External DNS Installed What we need to do ?
1. We need to manually add the DNS name in Azure DNS Zones before deploying "kube-manifests"
2. Get the Public IP from Azure Application Gateway -> agic-appgw -> Overview Tab
3. Go to DNS Zones -> Add Record set 
myapp1.kubeoncloud.com

# Verify Azure DNS
1. Go to DNS Zones -> kubeoncloud.com
2. Review the new DNS record created
```

## Step-06: Verify WAF Associated Application Gateways Section
```t
# Verify  WAF Associated Application Gateways Section
1. Go to Portal -> WAF -> myapp1-waf-policy1  
2. Settings -> Associated Application Gateways
3. We should see Route Path configured which means WAF Applied to Route paths only and not for complete Listener
```

## Step-07: Access Application 
```t
# Access Application (Local Desktop)
http://myapp1.kubeoncloud.com/app1/index.html
Observation:
1. This should not work
2. We blocked my internet public IP

# Access Application (Azure Cloud Shell)
http://myapp1.kubeoncloud.com/app1/index.html
Observation:
1. This should work
2. Azure cloud shell is not blocked in rule "BlockMyIPAddress" rule in WAF policy
```

## Step-08: Verify Application Gateway Logs
- [WAF Log Analytics Queries](https://learn.microsoft.com/en-us/azure/application-gateway/log-analytics)
```t
# AppGW Logs
1. Go to Portal -> agic-appgw -> Monitoring -> Logs
2. Review the logs using below queries

# Query-1: ApplicationGatewayAccess Log
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS" and OperationName == "ApplicationGatewayAccess"

# Query-2: ApplicationGatewayFirewallLog
AzureDiagnostics 
| where ResourceProvider == "MICROSOFT.NETWORK" and Category == "ApplicationGatewayFirewallLog"

# Query-3: Matched/Blocked requests by IP
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.NETWORK" and Category == "ApplicationGatewayFirewallLog"
| summarize count() by clientIp_s, bin(TimeGenerated, 1m)
| render timechart

# Query-4: Matched/Blocked requests by URI
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.NETWORK" and Category == "ApplicationGatewayFirewallLog"
| summarize count() by requestUri_s, bin(TimeGenerated, 1m)
| render timechart
```

## Step-09: Clean-Up Route Path Application
```t
# Delete Application
kubectl delete -f 01-kube-manifests-scope-RoutePath/
```

## Step-10: Review Kubernetes Manifests
- **kube-manifests folder:** 02-kube-manifests-scope-listener
- 01-NginxApp1-Deployment.yaml
- 02-NginxApp1-ClusterIP-Service.yaml

## Step-11: Review Ingress Manifest 
- When `Root Context` used WAF Policy associates to `Listeners`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-waf
  annotations:
    appgw.ingress.kubernetes.io/waf-policy-for-path: "/subscriptions/82808767-144c-4c66-a320-b30791668b0a/resourceGroups/agicdemo/providers/Microsoft.Network/ApplicationGatewayWebApplicationFirewallPolicies/myapp1-waf-policy1"
spec:
  ingressClassName: azure-application-gateway
  rules:
    - host: myapp1.kubeoncloud.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app1-nginx-clusterip-service
                port: 
                  number: 80
```

## Step-12: Deploy kube-manifests and verify
```t
# Deploy Application
kubectl apply -f 02-kube-manifests-scope-listener

# Verify WAF Associated Application Gateways section
1. Go to Portal -> WAF -> myapp1-waf-policy1  
2. Settings -> Associated Application Gateways
3. We should see "Listener" configured which means WAF Applied to "Listener" as we have used Root Contet "/" in our Ingress rules

# Access Application (Local Desktop)
http://myapp1.kubeoncloud.com/app1/index.html
Observation:
1. This should not work
2. We blocked my internet public IP

# Access Application (Azure Cloud Shell)
http://myapp1.kubeoncloud.com/app1/index.html
Observation:
1. This should work
2. Azure cloud shell is not blocked in rule "BlockMyIPAddress" rule in WAF policy
```

## Step-13: Clean-Up Route Path Application
```t
# Delete Application
kubectl delete -f 02-kube-manifests-scope-listener/
```