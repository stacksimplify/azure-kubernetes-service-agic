# AGIC - Build and Verify Backend SSL Application

## Step-01: Introduction


## Step-02: Backend - Create Self-Signed SSL Certificates 
```t
# Change Directory 
cd SSL-SelfSigned-Certs

# Create your backend private key:
openssl genrsa -out backend-ssl.key 2048

# Create your backend certificate signing request:
openssl req -new -key backend-ssl.key -out backend-ssl.csr -subj "/CN=backend.kubeoncloud.com"

# Create your backend certificate:
openssl x509 -req -days 7300 -in backend-ssl.csr -signkey backend-ssl.key -out backend-ssl.crt
```

## Step-03: Create Kubernetes Secret with  Backend Cert and Key
```t
# Change Directory 
cd SSL-SelfSigned-Certs

# Create a Secret that holds your backend certificate and key:
kubectl create secret tls backend-tls-secret  --cert backend-ssl.crt --key backend-ssl.key

# List Secrets
kubectl get secrets
```

## Step-04: Create Kubernetes Configmap for Nginx Config File
- **File Name:** 01-myapp-configmap.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configfile-cm
data:
  default.conf: |-
    server {
        listen 80 default_server;
        listen 443 ssl;
        root /usr/share/nginx/html;
        index index.html;
        ssl_certificate /etc/nginx/ssl/tls.crt;
        ssl_certificate_key /etc/nginx/ssl/tls.key;
    }
```

## Step-05: Create Kubernetes Deployment with VolumeMounts and Volumes
- **File Name:** 02-myapp-Deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-nginx-deployment
  labels:
    app: myapp-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp-nginx
  template:
    metadata:
      labels:
        app: myapp-nginx
    spec:
      containers:
        - name: myapp-nginx
          image: ghcr.io/stacksimplify/kubenginx:1.0.0
          ports:
            - containerPort: 443
          volumeMounts:
          - mountPath: /etc/nginx/ssl
            name: backend-ssl-certs-volume
          - mountPath: /etc/nginx/conf.d
            name: nginx-configfile-volume
      volumes:
      - name: backend-ssl-certs-volume
        secret:
          secretName: backend-tls-secret
      - name: nginx-configfile-volume
        configMap:
          name: nginx-configfile-cm
```

## Step-06: Create Kubernetes LoadBalancer Service
- **File Name:** 03-myapp-LoadBalancer-Service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-nginx-loadbalancer-service
  labels:
    app: myapp-nginx
spec:
  type: LoadBalancer
  selector:
    app: myapp-nginx
  ports:
    - port: 443
      targetPort: 443
```

## Step-07: Deploy and Verify
```t
# Deploy Application
kubectl deploy -f kube-manifests/

# List k8s Deployments
kubectl get deploy

# List k8s Services
kubectl get svc

# List k8s Pods
kubectl get pods

# Verify by connecting to Pod
kubectl exec --stdin --tty <POD-NAME> -- /bin/bash
kubectl exec --stdin --tty myapp-nginx-deployment-65b8578fcc-6qmpc -- /bin/bash

# Verify Nginx Config File in container 
cd /etc/nginx/conf.d
cat default.conf

# Verify SSL Certs in Container 
cd /etc/nginx/ssl/
cat tls.key
cat tls.crt

# Decode SSL Cert to verify it is the same SSL cert
https://www.sslshopper.com/certificate-decoder.html

# Access Application
kubectl get svc
https://<EXTERNAL-IP>
Observation:
1. Review SSL Certificate CN Name
2. It should match with what we created.
3. In our case it is "backend.kubeoncloud.com"
```

## Step-08: Clean-Up
```t
# Delete Application
kubectl delete -f kube-manifests/

# DONT Delete Kubernetes Secret
kubectl get secrets
Kubernetes Secret Name: backend-tls-secret  
Observation:
1. We will use this secret in next demo "E2E Ingress SSL"
```
