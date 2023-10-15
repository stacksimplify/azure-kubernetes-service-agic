# Ingress - Default Backend

## Step-01: Introduction
- We are going to understand and implement about Ingress Default Backend

## Step-02: Review k8s Application Manifests
- **Deployment Manifest:** 01-myapp-Deployment.yaml
- **Cluster IP Service Manifest:** 02-myapp-ClusterIP-Service.yaml

## Step-03: Review Ingress Service Manifests
- [Ingress API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/)
- **Ingress Manifest:** 03-Ingress-Default-Backend.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress-service
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway  
spec:
  defaultBackend:
    service:
      name: myapp-nginx-clusterip-service
      port: 
        number: 80
```

## Step-04: Deploy and Verify
```t
# Deploy Apps
kubectl apply -f kube-manifests/

# List Deployments
kubectl get deploy

# List Pods
kubectl get pods

# List Services
kubectl get svc

# List Ingress
kubectl get ingress
```

## Step-05: Access Applications
```t
# Access using curl
curl http://<INGRESS-IP>

# Access using Browser
http://<INGRESS-IP>
```

## Step-06: Verify Application Gateway Ingress Pod Logs
```t
# Verify ingress-appgw pod Logs 
kubectl -n kube-system logs -f $(kubectl -n kube-system get po | egrep -o 'ingress-appgw-deployment[A-Za-z0-9-]+')
```

## Step-07: Verify Azure AKS Cluster using Azure Portal
- **Workloads:**
  - We should see `myapp-nginx-deployment`
- **Services and Ingresses Tab:**
  - We should see `myapp-nginx-clusterip-service`

## Step-08: Verify Application Gateway using Azure Portal
- Please review and observe all default settings in detail
- **Backend Pools Tab:** 
  - You should see new pool with name like `pool-default-myapp-nginx-clusterip-service-80-bp-80`
  - Review `IP Address` in `Backend Targets`
  - Verify if it is same as our POD IP
- **Backend Settings Tab:**   
  - We should see a new setting with name as `bp-default-myapp-nginx-clusterip-service-80-80-myapp-ingress-service`
  - Review the probe which is `defaultprobe-Http`
- **Frontend IP configurations Tab:**  
  - We should see public ip configured
  - We should see private ip not configured
  - NO CHANGES
- **Listeners Tab:** 
  - We should see default port 80 listener configured
  - NO CHANGES
- **Rules Tab:** 
  - We should see default basic rule configured
  - Open default rule and review `Backend Targets` tab, you should see `pool-default-myapp-nginx-clusterip-service-80-bp-80`
  - `defaultaddresspool` changed to new pool created as part of our Kubernetes deployment
- **Health probes Tab:**       
  - We should see default HTTP and HTTPS probes
  - Review Pod logs
```t
# Verify pod Logs 
kubectl get pods 
kubectl logs -f <POD-NAME>
kubectl logs -f myapp-nginx-deployment-77df6b5877-wlznj  

# In parallel access Application via browser
http://<INGRESS-IP>

## HEALTH PROBE LOGS (Application Gateway polling the Pod)
10.225.0.6 - - [03/Sep/2023:11:33:27 +0000] "GET / HTTP/1.1" 200 218 "-" "-" "-"
10.225.0.4 - - [03/Sep/2023:11:33:28 +0000] "GET / HTTP/1.1" 200 218 "-" "-" "-"

# Log for request we made from browser
10.225.0.6 - - [03/Sep/2023:11:34:07 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/116.0.0.0 Safari/537.36" "175.101.19.211:19596"
```  

## Step-09: Clean-Up Applications
```t
# Delete Apps
kubectl delete -f kube-manifests/

# Review Application Gateway Tabs
1. Backend Pools Tab 
2. Backend Settings Tab
3. Rules Tab
```

