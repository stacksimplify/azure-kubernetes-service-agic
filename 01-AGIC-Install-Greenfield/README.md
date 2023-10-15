# Azure AGIC Install using Add-On on Azure AKS Cluster (Greenfield)

## Step-01: Introduction
- Enable the AGIC add-on for a new AKS cluster with a new application gateway instance
- Create New Azure AKS Cluster with 
  - New Application Gateway
  - **Add On:** AGIC (ingress-appgw)
  - Enable Autoscaling
 
## Step-02: Create AKS Cluster
```t
# Create Resource Group
az group create --location eastus --resource-group agicdemo

# Create AKS Cluser
az aks create --name agic-cluster \
              --resource-group agicdemo \
              --network-plugin azure \
              --enable-managed-identity \
              --enable-addons ingress-appgw \
              --appgw-name agic-appgw \
              --appgw-subnet-cidr "10.225.0.0/16" \
              --node-count 1 \
              --enable-cluster-autoscaler \
              --min-count 1 \
              --max-count 2 \
              --generate-ssh-keys 
```

## Step-03: Assign network contributor role to AGIC addon Managed Identity
```t
# Get application gateway id from AKS addon profile
appGatewayId=$(az aks show -n agic-cluster -g agicdemo -o tsv --query "addonProfiles.ingressApplicationGateway.config.effectiveApplicationGatewayId")
echo $appGatewayId

# Get Application Gateway subnet id
appGatewaySubnetId=$(az network application-gateway show --ids $appGatewayId -o tsv --query "gatewayIPConfigurations[0].subnet.id")
echo $appGatewaySubnetId

# Get AGIC addon identity
agicAddonIdentity=$(az aks show -n agic-cluster -g agicdemo -o tsv --query "addonProfiles.ingressApplicationGateway.identity.clientId")
echo $agicAddonIdentity

# Assign network contributor role to AGIC addon Managed Identity to subnet that contains the Application Gateway
az role assignment create --assignee $agicAddonIdentity --scope $appGatewaySubnetId --role "Network Contributor"
```

## Step-04: Verify AKS Cluster
```t
# configure kubeconfig 
az aks get-credentials --resource-group agicdemo --name agic-cluster

# List Kubernetes Nodes
kubectl get nodes
```

## Step-05: Verify AKS Add On 
```t
# List Kubernetes Deployments in kube-system namespace
kubectl get deploy -n kube-system
Observation:
1. Should find the deployment with name "ingress-appgw-deployment"
2. This is the Azure Application Gateway Ingress Controller Kubernetes Deployment Object

# List Pods
kubectl get pods -n kube-system

# Describe Pod
kubectl -n kube-system describe pod <AGIC-POD-NAME>
kubectl -n kube-system describe pod ingress-appgw-deployment-55965f45cf-x28fm  
Observation:
1. Review the line where you can find ingress current version downloaded
2. Pulling image "mcr.microsoft.com/azure-application-gateway/kubernetes-ingress:1.5.3"
3. You can also run the below command to find the AppGW Ingress version
kubectl get deploy ingress-appgw-deployment -o yaml -n kube-system | grep "image:"

# Verify ingress-appgw pod Logs
kubectl -n kube-system logs -f $(kubectl -n kube-system get po | egrep -o 'ingress-appgw-deployment[A-Za-z0-9-]+')
```

## Step-06: Verify latest release 
- [Azure AGIC Git Repo Releases](https://github.com/Azure/application-gateway-kubernetes-ingress/releases)
- What is our takeway?
  - We will always find AKS Addon will be using stable version which will be few versions lesser than the latest version present in `Releases` section
  - Azure recommends to use AKS Add On approach (Greenfield or Brownfield)
  - In production, we should prefer to use AKS Add On approach only
  - We can also use latest version of AGIC by deploying using HELM (Not Recommended for production)

## Step-07: Verify Azure AKS Cluster using Azure Portal
- Review AKS cluster all tabs

## Step-08: Verify Application Gateway using Azure Portal
- Please review and observe all default settings in detail
- **Overview Tab:**
  - Resource Group: MC_agicdemo_agic-cluster_eastus
  - Review frontend IP
  - Tier: V2
- **Configuration Tab:**
  - For cost saving we can also change instance count to 1 and save
  - In next steps we will also discuss about stopping Application Gateway using command line
- **Web Application Firewall Tab:** 
  - We will enable and use this in later demos of the course. 
- **Backend Pools Tab:** 
  - We should see default address pool
- **Backend Settings Tab:**   
  - We should see defaulthttpsetting
- **Frontend IP configurations Tab:**  
  - We should see public ip configured
  - We should see private ip not configured
- **Listeners Tab:** 
  - We should see default port 80 listener configured
- **Rules Tab:** 
  - We should see default basic rule configured
  - Open default rule and review `Backend Targets` tab, you should see `defaultaddresspool`
  - This changes when you deploy sample app in next demo. 
- **Health probes Tab:**       
  - We should see default HTTP and HTTPS probes
- **Properties Tab:**
  - Resource ID
  - Location
  - Resource group

## Step-09: Cost Saving - Stop Azure AKS and Application Gateway
```t
# Azure AKS Cluster
In portal, go to Overview Tab and click on "STOP"

# Azure Application Gateway STOP
az network application-gateway stop --name <APPGW-NAME> --resource-group <RESOURCE-GROUP>
az network application-gateway stop --name agic-appgw --resource-group MC_agicdemo_agic-cluster_eastus
```

## Step-10: Start Azure AKS and Application Gateway
```t
# Important Note
- Wait for 15 minutes before starting them back

# Azure AKS Cluster
In portal, go to Overview Tab and click on "START"

# Azure Application Gateway START
az network application-gateway start --name <APPGW-NAME> --resource-group <RESOURCE-GROUP>
az network application-gateway start --name agic-appgw --resource-group MC_agicdemo_agic-cluster_eastus
```


## Additional References
- [Greenfield: AGIC Add-On on AKS Cluster](https://learn.microsoft.com/en-us/azure/application-gateway/tutorial-ingress-controller-add-on-new)
- [Brownfield: AGIC Add-On on AKS Cluster](https://learn.microsoft.com/en-us/azure/application-gateway/tutorial-ingress-controller-add-on-existing)
- [Greenfield: AGIC using Helm on AKS Cluster](https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-install-new)
- [Brownfield: AGIC using Helm on AKS Cluster](https://learn.microsoft.com/en-us/azure/application-gateway/ingress-controller-install-existing)
- https://azure.microsoft.com/en-us/blog/application-gateway-ingress-controller-for-azure-kubernetes-service/
- https://learn.microsoft.com/en-us/azure/application-gateway/tutorial-ingress-controller-add-on-new?tryIt=true&source=docs#deploy-an-aks-cluster-with-the-add-on-enabled  
- https://learn.microsoft.com/en-us/azure/architecture/example-scenario/aks-agic/aks-agic
