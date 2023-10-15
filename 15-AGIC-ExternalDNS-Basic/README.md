# Azure AGIC Ingress - External DNS Basic

## Step-01: Introduction
- Deploy a sample Application with Ingress in combination with External DNS
- When we use External DNS, Hostname defined in `Ingress Service` should be automatically added as `Record Set` in `Azure DNS Zones`

## Step-02: Review Kubernetes Manifests
- 01-NginxApp1-Deployment.yaml
- 02-NginxApp1-ClusterIP-Service.yaml
- 03-Ingress-with-ExternalDNS.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginxapp1-ingress-service
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

## Step-03: Deploy Application and Test

### Step-03-01: Deploy Application
```t
# Deploy Application
kubectl apply -f kube-manifests/

# Verify Pods and Services
kubectl get po,svc

# Verify Ingress
kubectl get ingress
```

### Step-03-02: Verify logs in External DNS Pod
- Wait for 1 to 3 minutes for Record Set update in DNZ Zones
```t
# Verify ExternalDNS Logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
```
- External DNS Pod Logs
```log
time="2023-09-01T06:21:58Z" level=info msg="Updating A record named 'myapp1' to '13.86.123.83' for Azure DNS zone 'kubeoncloud.com'."
time="2023-09-01T06:21:59Z" level=info msg="Updating TXT record named 'externaldns-myapp1' to '\"heritage=external-dns,external-dns/owner=default,external-dns/resource=ingress/default/nginxapp1-ingress-service\"' for Azure DNS zone 'kubeoncloud.com'."
time="2023-09-01T06:22:00Z" level=info msg="Updating TXT record named 'externaldns-a-myapp1' to '\"heritage=external-dns,external-dns/owner=default,external-dns/resource=ingress/default/nginxapp1-ingress-service\"' for Azure DNS zone 'kubeoncloud.com'."
```

### Step-03-03: Verify Record Set in DNS Zones -> kubeoncloud.com
- Go to All Services -> DNS Zones -> kubeoncloud.com
- Verify if we have `myapp1.kubeoncloud.com` created
```t
# Template Command
az network dns record-set a list -g <Resource-Group-dnz-zones> -z <yourdomain.com>

# Replace DNS Zones Resource Group and yourdomain
az network dns record-set a list -g dns-zones -z kubeoncloud.com

# Additionally you can review via Azure Portal
Go to Portal -> DNS Zones -> <YOUR-DOMAIN>
Review records in "Overview" Tab
```
- Perform `nslookup` test
```t
# nslookup Test
Kalyans-Mac-mini:kube-manifests kalyanreddy$ nslookup  myapp1.kubeoncloud.com
Server:		192.168.1.1
Address:	192.168.1.1#53

Non-authoritative answer:
Name:	myapp1.kubeoncloud.com
Address: 52.158.161.63

Kalyans-Mac-mini:kube-manifests kalyanreddy$ 

```

### Step-03-04: Access Application and Test
```t
# Access Application
http://myapp1.kubeoncloud.com
http://myapp1.kubeoncloud.com/app1/index.html

# Important Note: Replace kubeoncloud.com with your domain name
```

## Step-04: Clean-Up
### Step-04-01: Delete Application and Verify
```t
# Delete Application
kubectl delete -f kube-manifests/

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
### Step-04-02: Deleting DNS Record Log from External DNS
```log
time="2023-09-01T06:22:59Z" level=info msg="Deleting A record named 'myapp1' for Azure DNS zone 'kubeoncloud.com'."
time="2023-09-01T06:23:00Z" level=info msg="Deleting TXT record named 'externaldns-myapp1' for Azure DNS zone 'kubeoncloud.com'."
time="2023-09-01T06:23:00Z" level=info msg="Deleting TXT record named 'externaldns-a-myapp1' for Azure DNS zone 'kubeoncloud.com'."
```