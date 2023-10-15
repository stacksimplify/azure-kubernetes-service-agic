# Ingress - Paths

## Step-01: Introduction
- We are going to understand and implement about `Ingress Paths`
- In addition, also discuss about `spec.ingressClassName`
```t
# List Ingress Classes on AKS Cluster
kubectl get ingressclass 

# Describe Ingress Class
kubectl describe ingressclass <INGRESS-CLASS-NAME>
kubectl describe ingressclass azure-application-gateway

# Review YAM Output of Ingress Class
kubectl get ingressclass azure-application-gateway -o yaml
```

## Step-02: Review k8s Application Manifests
- **App1 Deployment Manifest:** 01-NginxApp1-Deployment.yaml
- **App1 Cluster IP Service Manifest:** 02-NginxApp1-ClusterIP-Service.yaml

## Step-03: Review Ingress Service Manifests
- [Ingress API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/)
- Discuss in detail about [Ingress Paths](https://kubernetes.io/docs/concepts/services-networking/ingress/#examples)
- **Ingress Manifest:** 03-Ingress-HTTP-Paths.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginxapp1-ingress-service
spec:
  ingressClassName: azure-application-gateway
  rules:
    - http:
        paths:
          - path: /       # Comment at Step-09
          #- path: /app1  # UnComment at Step-09
            pathType: Prefix
            backend:
              service:
                name: app1-nginx-clusterip-service
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
# Access Root Context
curl http://<INGRESS-IP>
http://<INGRESS-IP>
Observation: default nginx index.html should be displayed

# Access /app1
curl http://<INGRESS-IP>/app1/index.html
http://<INGRESS-IP>/app1/index.html
Observation: app1 index.html should be displayed
```

## Step-06: Verify Application Gateway Ingress Pod Logs
```t
# Verify ingress-appgw pod Logs 
kubectl -n kube-system logs -f $(kubectl -n kube-system get po | egrep -o 'ingress-appgw-deployment[A-Za-z0-9-]+')
```

## Step-07: Verify Azure AKS Cluster using Azure Portal
- **Workloads:**
  - We should see `app1-nginx-deployment`
- **Services and Ingresses Tab:**
  - We should see `app1-nginx-clusterip-service`

## Step-08: Verify Application Gateway using Azure Portal
- Please review and observe all default settings in detail
- **Backend Pools Tab:** Review Backend Pools
- **Backend Settings Tab:**   
  - Review `Backend Port`
  - Review the probe `custom probe`
- **Rules Tab:** 
  - We should see default basic rule configured
  - Open default rule and review `Backend Targets` tab
- **Health probes Tab:**       
  - We should see `custom HTTP probe` created
  - Review Pod logs (optional)
```t
# Verify pod Logs 
kubectl get pods 
kubectl logs -f <POD-NAME>
kubectl logs -f app1-nginx-deployment-ff56dcf94-j7nq8

# In parallel access Application via browser
http://<INGRESS-IP>/app1/index.html

## HEALTH PROBE LOGS (Application Gateway polling the Pod)
10.225.0.6 - - [03/Sep/2023:11:33:27 +0000] "GET / HTTP/1.1" 200 218 "-" "-" "-"
10.225.0.4 - - [03/Sep/2023:11:33:28 +0000] "GET / HTTP/1.1" 200 218 "-" "-" "-"

# Log for request we made from browser
10.225.0.6 - - [03/Sep/2023:11:34:07 +0000] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/116.0.0.0 Safari/537.36" "175.101.19.211:19596"
```  

## Step-09: Update Ingress Path, Deploy and Verify
```t
# Change Path from / to /app1
spec:
  rules:
    - http:
        paths:
          #- path: /       # Comment at Step-09
          - path: /app1   # UnComment at Step-09

# Deploy and Verify
kubectl apply -f kube-manifests

# Access Root Context
curl http://<INGRESS-IP>
http://<INGRESS-IP>
Observation: 
1. This should fail, and we should get 502 bad gateway error from AppGw
2. Go to AppGw -> Rules -> Backend Targets, we should see a path based rule

# Access /app1
curl http://<INGRESS-IP>/app1/index.html
http://<INGRESS-IP>/app1/index.html
Observation: 
1. app1 index.html should be displayed
2. Go to AppGw -> Rules -> Backend Targets, we should see a path based rule
```

## Step-10: Scale the Application and Verify if new pods added in AppGw Backend Pools
```t
# List Pod with Pod IP
kubectl get pods -o wide

# Scale our Application and see those all pod IPs displayed as Backend Targets
kubectl scale deployment app1-nginx-deployment --replicas=5

# List Pod with Pod IP
kubectl get pods -o wide

# Go to AppGw -> Backend Pools
1. We should see 5 pods added with Pod IPs
```  

## Step-11: Clean-Up Applications
```t
# Delete Apps
kubectl delete -f kube-manifests/

# Review Application Gateway Tabs
1. Backend Pools Tab 
2. Backend Settings Tab
3. Rules Tab
```