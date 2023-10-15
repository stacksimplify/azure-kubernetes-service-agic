# AGIC WAF - OWASP Default Policies

## Step-01: Introduction
- Enable WAFV2 for Application Gateway
- Enable default OWASP rules in WAFV2
- Enable Application Gateway Logs (Send to Log Analytics Workspace)
- Deploy simple k8s Application 
- Verify if default OWASP rules working as expected with WAF for our application
- Verify Application Gateway Access and WAF Firewall logs using Queries

## Step-02: Enable WAF on Azure Application Gateway
```t
# Enable WAF on Azure Application Gateway
Go to Portal -> Application Gateway -> agic-appgw -> Settings

# AppGW Configuration
Tier: WAF V2
Click on Save

# AppGW Web Application Firewall
## Configure Tab
Tier: WAF V2
WAF Status: Enabled
WAF Mode: Prevention

## Rules Tab
Rules Set: OWASP 3.2
Click on Save
```

## Step-03: Create Log Analytics Workspace
```t
# Create Log Analytics Workspace
1. Go to Portal -> Log Analytics Workspaces -> create
2. Resource group: agicdemo
3. Name: agic-log-analytics-workspace
4. Region: East US
5. Review + Create
```

## Step-04: Enable Diagnostic Settings in Azure Application Gateway
```t
# Enable Diagnostic Settings in AppGW
1. Go to Portal -> AppGW -> agic-appgw -> Monitoring -> Diagnostic settings
2. Category groups: all logs
3. Metrics: All Metrics
4. Destination details: Send to Log Analytics workspace
5. Subscription: YOUR-SUBSCRIPTIOn
6. Log Analytics workspace: agic-log-analytics-workspace
7. Click on "Save"
```

## Step-05: Review Kubernetes Manifests
- 01-NginxApp1-Deployment.yaml
- 02-NginxApp1-ClusterIP-Service.yaml
- 03-Ingress-WAF.yaml

## Step-06: Deploy Kubernetes Manifests and Verify
```t
# Change Directory
cd 23-AGIC-WAF-OWASP

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

## Step-07: Access Application 
```t
# Access Application
http://myapp1.kubeoncloud.com/app1/index.html
Observation:
1. This should work

# Access Application (XSS Attack URL)
http://myapp1.kubeoncloud.com/app1/index.html?"<script>show tables</script>"
Observation:
1. This should throw "403 forbidden" error from "Microsoft-Azure-Application-Gateway/v2"
2. WAF blocked the XSS Attack, didnt send the request to next level
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

Query-3: Matched/Blocked requests by IP
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.NETWORK" and Category == "ApplicationGatewayFirewallLog"
| summarize count() by clientIp_s, bin(TimeGenerated, 1m)
| render timechart

Query-4: Matched/Blocked requests by URI
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.NETWORK" and Category == "ApplicationGatewayFirewallLog"
| summarize count() by requestUri_s, bin(TimeGenerated, 1m)
| render timechart
```

## Step-09: Clean-Up
```t
# Delete Application
kubectl delete -f kube-manifests/
```