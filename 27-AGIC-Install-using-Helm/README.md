# AGIC Install using Helm

## Step-01: Introduction
- Create Azure AKS Cluster
- Create Azure Application Gateway
- Deploy Azure AGIC using Helm

## Step-02: Create Resource Group, Virtual Network and Subnets 
```t
# Set Environment Variables
export RESOURCE_GROUP_NAME="agic-helm"
export VNET_NAME="agic-vnet"
export AKS_SUBNET_NAME="aks-subnet"
export APPLICATION_GATEWAY_SUBNET_NAME="appgw-subnet"
export AKS_CLUSTER_NAME="agic-cluster-helm"
export APPLICATION_GATEWAY_PUBLICIP_NAME="appgw-public-ip"
export APPLICATION_GATEWAY_NAME="agic-appgw-helm"
export USER_ASSIGNED_IDENTITY_NAME="agic-identity-helm"
export FEDERATED_IDENTITY_CREDENTIAL_NAME="fed-identity-helm"

# Create Resource Group
az group create --name "${RESOURCE_GROUP_NAME}" --location eastus

# Create Virtual Network
az network vnet create --resource-group "${RESOURCE_GROUP_NAME}" --name "${VNET_NAME}" --address-prefixes 10.224.0.0/12

# Create Azure AKS Subnet
az network vnet subnet create --resource-group "${RESOURCE_GROUP_NAME}" --vnet-name "${VNET_NAME}" --name "${AKS_SUBNET_NAME}" --address-prefixes 10.224.0.0/16

# Create Azure Application Gateway Subnet
az network vnet subnet create --resource-group "${RESOURCE_GROUP_NAME}" --vnet-name "${VNET_NAME}" --name "${APPLICATION_GATEWAY_SUBNET_NAME}" --address-prefixes 10.225.0.0/16
```

## Step-03: Create Azure AKS Cluster
```t
# Export AKS Subnet ID
export AKS_SUBNET_ID="$(az network vnet subnet show --resource-group "${RESOURCE_GROUP_NAME}" --vnet-name "${VNET_NAME}" --name "${AKS_SUBNET_NAME}" --query id --output tsv)"
echo $AKS_SUBNET_ID

# Create Azure AKS Cluster
az aks create -g "${RESOURCE_GROUP_NAME}" -n "${AKS_CLUSTER_NAME}" --node-count 1 --enable-oidc-issuer --enable-workload-identity --network-plugin azure --vnet-subnet-id "${AKS_SUBNET_ID}" --enable-cluster-autoscaler --min-count 1 --max-count 2 --generate-ssh-keys

# Configure kubeconfig for kubectl
az aks get-credentials --resource-group ${RESOURCE_GROUP_NAME} --name ${AKS_CLUSTER_NAME}

# List Kubernetes Nodes
kubectl get nodes
```

## Step-04: Create Azure Application Gateway
```t
# Create Public IP
az network public-ip create -g "${RESOURCE_GROUP_NAME}" -n "${APPLICATION_GATEWAY_PUBLICIP_NAME}" --allocation-method Static --sku Standard --tier Regional

# Create Azure Application Gateway
az network application-gateway create -g "${RESOURCE_GROUP_NAME}" -n "${APPLICATION_GATEWAY_NAME}" --sku Standard_v2 --public-ip-address "${APPLICATION_GATEWAY_PUBLICIP_NAME}" --vnet-name "${VNET_NAME}" --subnet "${APPLICATION_GATEWAY_SUBNET_NAME}" --priority 105
```

## Step-05: Create User Assigned Managed Identity
```t
# Create User Assigned Managed Identity
az identity create --name "${USER_ASSIGNED_IDENTITY_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" 

# export the ClientID of the User Assigned Managed Identity
export USER_ASSIGNED_IDENTITY_CLIENT_ID="$(az identity show --resource-group "${RESOURCE_GROUP_NAME}" --name "${USER_ASSIGNED_IDENTITY_NAME}" --query 'clientId' -otsv)"
echo $USER_ASSIGNED_IDENTITY_CLIENT_ID
```

## Step-06: Create Federated Identity Credential
```t
# Export the Azure AKS OIDC Issuer URL
export AKS_OIDC_ISSUER="$(az aks show -n "${AKS_CLUSTER_NAME}" -g "${RESOURCE_GROUP_NAME}" --query "oidcIssuerProfile.issuerUrl" -otsv)"
echo $AKS_OIDC_ISSUER

# Create Federated Identity Credential
az identity federated-credential create --resource-group ${RESOURCE_GROUP_NAME} --name ${FEDERATED_IDENTITY_CREDENTIAL_NAME} --identity-name ${USER_ASSIGNED_IDENTITY_NAME} --issuer ${AKS_OIDC_ISSUER} --subject system:serviceaccount:default:ingress-azure
```

## Step-07: Create a Contributor Role for the Managed Identity over the Application Gateway
```t
# Export Application Gateway Resource ID
export APPLICATION_GATEWAY_RESOURCE_ID="$(az network application-gateway show --name "${APPLICATION_GATEWAY_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" --query 'id' --output tsv)"
echo $APPLICATION_GATEWAY_RESOURCE_ID

# Create Contributor Role for the Managed Identity over the Application Gateway
az role assignment create --assignee "${USER_ASSIGNED_IDENTITY_CLIENT_ID}" --scope "${APPLICATION_GATEWAY_RESOURCE_ID}" --role Contributor
```

## Step-08: Create a Contributor Role for the Managed Identity over the Virtual Network
```t
# Export Virtual Network Resource ID
export VIRTUAL_NETWORK_RESOURCE_ID="$(az network vnet show --name "${VNET_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" --query 'id' --output tsv)"
echo $VIRTUAL_NETWORK_RESOURCE_ID

# Create Contributor Role for the Managed Identity over the Virtual Network
az role assignment create --assignee "${USER_ASSIGNED_IDENTITY_CLIENT_ID}" --scope "${VIRTUAL_NETWORK_RESOURCE_ID}" --role Contributor
```

## Step-09: Create a Contributor Role for the Managed Identity over the AGIC Public IP
```t
# Export AGIC Public IP Resource ID
export APPLICATION_GATEWAY_PUBLICIP_RESOURCE_ID="$(az network public-ip show --name "${APPLICATION_GATEWAY_PUBLICIP_NAME}" --resource-group "${RESOURCE_GROUP_NAME}" --query 'id' --output tsv)"
echo $APPLICATION_GATEWAY_PUBLICIP_RESOURCE_ID

# Create Contributor Role for the Managed Identity over the AGIC Public IP
az role assignment create --assignee "${USER_ASSIGNED_IDENTITY_CLIENT_ID}" --scope "${APPLICATION_GATEWAY_PUBLICIP_RESOURCE_ID}" --role Contributor
```

## Step-10: Review helm-config.yaml 
- [helm-config.yaml](https://github.com/Azure/application-gateway-kubernetes-ingress/blob/master/docs/examples/sample-helm-config.yaml)
- Update the below details in `helm-config.yaml`
```t
# Get Subscription ID
subscriptionId="$(az account list --query "[?isDefault].id" -o tsv)"
echo $subscriptionId

# Get Application Gateway Resource Group and Application Gatewau Name
echo $RESOURCE_GROUP_NAME
echo $APPLICATION_GATEWAY_NAME

# Get User Assigned Managed Identity Client ID
# export the ClientID of the User Assigned Managed Identity
export USER_ASSIGNED_IDENTITY_CLIENT_ID="$(az identity show --resource-group "${RESOURCE_GROUP_NAME}" --name "${USER_ASSIGNED_IDENTITY_NAME}" --query 'clientId' -otsv)"
echo $USER_ASSIGNED_IDENTITY_CLIENT_ID
```

## Step-11: helm-config.yaml
```yaml
# This file contains the essential configs for the ingress controller helm chart

# Verbosity level of the App Gateway Ingress Controller
verbosityLevel: 3

################################################################################
# Specify which application gateway the ingress controller will manage
#
appgw:
    subscriptionId: 82808767-144c-4c66-a320-b30791668b0a
    resourceGroup: agic-helm
    name: agic-appgw-helm
    usePrivateIP: false

    # Setting appgw.shared to "true" will create an AzureIngressProhibitedTarget CRD.
    # This prohibits AGIC from applying config for any host/path.
    # Use "kubectl get AzureIngressProhibitedTargets" to view and change this.
    shared: true

################################################################################
# Specify which kubernetes namespace the ingress controller will watch
# Default value is "default"
# Leaving this variable out or setting it to blank or empty string would
# result in Ingress Controller observing all acessible namespaces.
#
# kubernetes:
#   watchNamespace: <namespace>

################################################################################
# Specify the authentication with Azure Resource Manager
#
# Two authentication methods are available:
# - Option 1: AAD-Pod-Identity (https://github.com/Azure/aad-pod-identity)
armAuth:
    type: workloadIdentity
    identityClientID: 81564d2c-561f-4c17-aabd-9491d7094d01

## Alternatively you can use Service Principal credentials
# armAuth:
#    type: servicePrincipal
#    secretJSON: <<Generate this value with: "az ad sp create-for-rbac --subscription <subscription-uuid> --sdk-auth | base64 -w0" >>

################################################################################
# Specify if the cluster is RBAC enabled or not
rbac:
    enabled: true # true/false
```

## Step-12: Install AGIC using Helm CLI
```t
# Upload helm-config.yaml to Azure Cloud Shell
Azure Cloud Shell -> Upload -> helm-config.yaml

# List Helm repositories
helm repo list

# Add Helm Repository
helm repo add application-gateway-kubernetes-ingress https://appgwingress.blob.core.windows.net/ingress-azure-helm-package/
helm repo update

# List Helm repositories
helm repo list

# Search Helm Repo (latest chart versions)
helm search repo application-gateway-kubernetes-ingress 

# Search Helm Repo (All Chart Versions)
helm search repo -l application-gateway-kubernetes-ingress --versions

# Upload helm-config.yaml to Azure Cloud Shell
helm-config.yaml

# Install Helm Release
helm install ingress-azure \
  -f helm-config.yaml \
  application-gateway-kubernetes-ingress/ingress-azure \
  --version 1.7.2
```

## Step-13: Verify the Installed Helm Release
```t
# List Helm Releases
helm list

# Verify all Kubernetes Resources created as part of this Helm Release
helm status <RELEASE-NAME> --show-resources
helm status ingress-azure --show-resources

# List Resources using kubectl
kubectl get deploy
kubectl get pods
kubectl get configmap
kubectl get clusterrole
kubectl get clusterrolebinding
kubectl get ingressclass

# Verify Service Accounts (Get and Describe)
kubectl get serviceaccount
kubectl describe serviceaccount ingress-azure
Observation:
1. Review the Annotations section where we find "azure.workload.identity/client-id:XXXXXX"
```
## Step-14: Review and Deploy Sample Application with Ingress Service
- 01-NginxApp1-Deployment.yaml
- 02-NginxApp1-ClusterIP-Service.yaml
- 03-Ingress-HTTP-Paths.yaml
```t
# Deploy Kubernetes Manifests
kubectl apply -f kube-manifests

# List Kubernetes Deployments
kubectl get deploy

# List Kubernetes Pods
kubectl get pods

# List Kubernetes Services
kubectl get svc

# List Kubernetes Ingress Service
kubectl get ingress

# Access Application
http://APPGW-PUBLICIP/app1/index.html
```

## Step-15: Clean-Up
```t
# Delete Kubernetes Applications
kubectl delete -f kube-manifests
```

