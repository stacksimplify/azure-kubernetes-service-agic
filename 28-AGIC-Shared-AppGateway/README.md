# AGIC - Shared Application Gateway

## Step-01: Introduction
1. Learn to implement on using AGIC in combination with Shared Application Gateway
2. In this demo we are going to share Azure Application Gateway with two other Azure Container Instances in addition to  AKS Apps

## Step-02: Update "shared: true" in helm-config.yaml 
- **File:** helm-config-shared-true.yaml
```yaml
appgw:
    subscriptionId: 82808767-144c-4c66-a320-b30791668b0a
    resourceGroup: agic-helm
    name: agic-appgw-helm
    usePrivateIP: false

    # Setting appgw.shared to "true" will create an AzureIngressProhibitedTarget CRD.
    # This prohibits AGIC from applying config for any host/path.
    # Use "kubectl get AzureIngressProhibitedTargets" to view and change this.
    shared: true
```

## Step-03: Upgrade Helm Release and Verify
```t
# List Helm Releases
helm list

# Upgrade Helm Release 
helm upgrade ingress-azure application-gateway-kubernetes-ingress/ingress-azure -f helm-config-shared-true.yaml

# List Helm Releases
helm list
Observation:
1. Helm release will be updated with new revision

# List Kubernetes Pods
kubectl get pods

# List Prohibited Targets
kubectl get AzureIngressProhibitedTargets
Observation:
1. Setting appgw.shared to "true" will create an AzureIngressProhibitedTarget CRD.
2. This prohibits AGIC from applying config for any host/path.

# Describe Prohibited Target
kubectl describe AzureIngressProhibitedTargets prohibit-all-targets

# YAML Output for Prohibited Target
kubectl get AzureIngressProhibitedTargets prohibit-all-targets -o yaml
```

## Step-04: Deploy Ingress Service and Verify on Application Gateway
```t
# Deploy Kubernetes Application
kubectl apply -f 01-kube-manifests
Observation:
1. "prohibit-all-targets" prohibits AGIC from applying config for any host/path.
2. Ingress Service created in AKS cluster but AGIC will not be able to populate the equivalent config on Azure Application Gateway

# Clean-Up
kubectl delete -f 01-kube-manifests
```

## Step-05: myaciapp1: Create Azure Container Instance App and Configure it in Application Gateway
- We are going to share our Azure Application Gateway with Azure Container Instances 

### Step-05-01: Create myaciapp1 Container Instance
```t
# Create myaciapp1
#### BASICS TAB
Resource Group: agic-helm
Container Name: myaciapp1
Region: East US
Availability Zones: NONE (LEAVE TO DEFAULT)
SKU: Standard (LEAVE TO DEFAULT)
Image Source: Other Registry
Image Type: Public
Image: ghcr.io/stacksimplify/kube-nginxapp1:1.0.0
OS Type: Linux
Size: 1 vcpu, 1.5 GB Memory (LEAVE TO DEFAULTS)

#### Networking
Networking Type: Public
DNS Label: myaciapp1
DNS Label scope reuse: Tenant (LEAVE TO DEFAULTS)
Ports: 80

#### ADVANCED TAB
LEAVE TO DEFAULTS

#### TAGS TAB
LEAVE TO DEFAULTS

REVIEW AND CREATE
CREATE
```

### Step-05-02: Create myaciapp1 Application Gateway Configurations
```t
# Application Gateway: agic-appgw-helm
## Backend Pool
Name: myaciapp1-bpool
Target: myaciapp1.ejaxevaacmbzhegf.eastus.azurecontainer.io
LEAVE ALL TO DEFAULTS AND CREATE

## Backend Settings
Name: myaciapp1-bset
LEAVE ALL DEFAULTS AND CREATE

## Listener
Name: myaciapp1-listener
Listener Type: Multisite
Host type: Multiple/wildcard
Host names: myaciapp1.kubeoncloud.com
LEAVE ALL TO DEFAULTS AND CREATE

## Rules
Name: myaciapp1-rule
Priority: 100
Listener: myaciapp1-listener
Target Type: Backend Pool
Backend Target: myaciapp1-bpool
Backend Settings: myaciapp1-bset
LEAVE ALL TO DEFAULTS AND CREATE
```

### Step-05-03: Create DNS Record in Azure DNS Zones and Access Application
```t
# DNS Zones: kubeoncloud.com
Name: myaciapp1
IP Address: PUBLIC-IP of AppGw 
Final DNS Name looks like: myaciapp1.kubeoncloud.com

# Verify Application
http://myaciapp1.kubeoncloud.com/app1/index.html
```


## Step-06: myaciapp2: Create Azure Container Instance App and Configure it in Application Gateway
- We are going to share our Azure Application Gateway with Azure Container Instances 

### Step-06-01: Create myaciapp1 Container Instance
```t
# Create myaciapp2
#### BASICS TAB
Resource Group: agic-helm
Container Name: myaciapp2
Region: East US
Availability Zones: NONE (LEAVE TO DEFAULT)
SKU: Standard (LEAVE TO DEFAULT)
Image Source: Other Registry
Image Type: Public
Image: ghcr.io/stacksimplify/kube-nginxapp2:1.0.0
OS Type: Linux
Size: 1 vcpu, 1.5 GB Memory (LEAVE TO DEFAULTS)

#### Networking
Networking Type: Public
DNS Label: myaciapp1
DNS Label scope reuse: Tenant (LEAVE TO DEFAULTS)
Ports: 80

#### ADVANCED TAB
LEAVE TO DEFAULTS

#### TAGS TAB
LEAVE TO DEFAULTS

REVIEW AND CREATE
CREATE
```

### Step-06-02: Create myaciapp2 Application Gateway Configurations
```t
# Application Gateway: agic-appgw-helm
## Backend Pool
Name: myaciapp2-bpool
Target: myaciapp1.ejaxevaacmbzhegf.eastus.azurecontainer.io
LEAVE ALL TO DEFAULTS AND CREATE

## Backend Settings
Name: myaciapp2-bset
LEAVE ALL DEFAULTS AND CREATE

## Listener
Name: myaciapp2-listener
Listener Type: Multisite
Host type: Multiple/wildcard
Host names: myaciapp2.kubeoncloud.com
LEAVE ALL TO DEFAULTS AND CREATE

## Rules
Name: myaciapp2-rule
Priority: 100
Listener: myaciapp2-listener
Target Type: Backend Pool
Backend Target: myaciapp2-bpool
Backend Settings: myaciapp2-bset
LEAVE ALL TO DEFAULTS AND CREATE
```

### Step-06-03: Create DNS Record in Azure DNS Zones and Access Application
```t
# DNS Zones: kubeoncloud.com
Name: myaciapp2
IP Address: PUBLIC-IP of AppGw 
Final DNS Name looks like: myaciapp2.kubeoncloud.com

# Verify Application
http://myaciapp2.kubeoncloud.com/app2/index.html
```

## Step-07: Deploy prohibit-myaciapp1 AzureIngressProhibitedTarget and Remove prohibit-all-targets
```t
# List AzureIngressProhibitedTargets
kubectl get AzureIngressProhibitedTargets 

# Deply prohibit-myaciapp1 AzureIngressProhibitedTarget
kubectl apply -f 02-prohibit-files/prohibit-myaciapp1.yaml
Observation:
1. We will ensure AGIC prohibits touching myaciapp1 configs
2. We will not create any prohibit for myaciapp2 and see how AGIC clears all the myaciapp2 configs from Azure Application Gateway

# List AzureIngressProhibitedTargets
kubectl get AzureIngressProhibitedTargets 

# Delete prohibit-all-targets
kubectl delete AzureIngressProhibitedTargets prohibit-all-targets
```

## Step-08: Deploy Sample Application and see what happens in Azure Application Gateway
```t
# Deploy k8s Application
kubectl apply -f 01-kube-manifests
Observation:
1. AGIC creates myapp configs in AppGw
2. AGIC doesnt touch myaciapp1 configs in AppGw because we created "prohibit-myaciapp1"
3. AGIC deletes myaciapp2 configs in AppGw
```

## Step-09: Clean-Up
```t
# Delete k8s Applications
kubectl delete -f 01-kube-manifests

# Delete myaciapp1 and myaciapp2
1. Delete myaciapp1 Container Instance
2. Delete myaciapp2 Container Instance

# Cleanup myaciapp1 related configs in AppGw
1. Listener
2. Rule
3. Backend Setting
4. Backend Pool
```

