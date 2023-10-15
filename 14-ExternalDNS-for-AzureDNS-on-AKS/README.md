# Kubernetes ExternalDNS to create Record Sets in Azure DNS from AKS

## Step-01: Introduction
- Create External DNS Manifest
- Provide Access to Azure DNS Zones using **Azure Managed Service Identity** for External DNS pod to create **Record Sets** in Azure DNS Zones
- Review Application & Ingress Manifests
- Deploy and Test


## Step-02: Create External DNS Manifests
- External-DNS needs permissions to Azure DNS to modify (Add, Update, Delete DNS Record Sets)
- We can provide permissions to External-DNS pod in two ways in Azure 
  - Using Azure Service Principal
  - Managed identity using AKS Kubelet identity
- We are going to use `Managed Identity` for providing necessary permissions here which is latest and greatest in Azure as on today. 

## Step-03: Fetching the Kubelet identity
```t
# Option-1: Get PRINCIPAL_ID using Commands
CLUSTER_GROUP=agicdemo
CLUSTERNAME=agic-cluster
PRINCIPAL_ID=$(az aks show --resource-group $CLUSTER_GROUP --name $CLUSTERNAME --query "identityProfile.kubeletidentity.objectId" --output tsv)
echo $PRINCIPAL_ID 

# Option-2: Get PRINCIPAL_ID using Azure Portal
1. Go to Resource Groups -> MC_agicdemo_agic-cluster_eastus
2. Find Managed Identity Resource with name "agic-cluster-agentpool"
3. Make a note of "Object (principal) ID"
PRINCIPAL_ID=5657b353-282c-4208-93a0-c99ac23a4ef3
echo $PRINCIPAL_ID 
```

## Step-04: Assign rights for the Kubelet identity
- Grant access to Azure DNS zone for the kublet identity.
```t
## Option-1: Get DNS_ID using Command Line
# DNS zone name like example.com or sub.example.com
AZURE_DNS_ZONE="kubeoncloud.com" 

# resource group where DNS zone is hosted
AZURE_DNS_ZONE_RESOURCE_GROUP="dns-zones" 

# Option-1: fetch DNS id used to grant access to the kublet identity
DNS_ID=$(az network dns zone show --name $AZURE_DNS_ZONE --resource-group $AZURE_DNS_ZONE_RESOURCE_GROUP --query "id" --output tsv)
echo $DNS_ID

## Option-2: Get DNS_ID using Azure Portal
1. Go to Azure Portal -> DNS Zones -> YOURDOMAIN.COM (kubeoncloud.com)
2. Go to Properties
3. Make a note of "Resource ID"
DNS_ID=/subscriptions/82808767-144c-4c66-a320-b30791668b0a/resourceGroups/dns-zones/providers/Microsoft.Network/dnszones/kubeoncloud.com
echo $DNS_ID

# Grant access to Azure DNS zone for the kubelet identity.
az role assignment create --role "DNS Zone Contributor" --assignee $PRINCIPAL_ID --scope $DNS_ID

# Verify the Azure Role Assignment in Managed Identities
1. Go to Azure Portal -> Managed Identities -> agic-cluster-agentpool
2. Go to Azur role assignments Tab
3. We can see "DNS Zone Contributor" role assigned to "agic-cluster-agentpool" Managed Identity
```

## Step-05: Gather Information Required for azure.json file
```t
# To get Azure Tenant ID
az account show --query "tenantId"

# To get Azure Subscription ID
az account show --query "id"
```

## Step-06: Make a note of Client Id and update in azure.json
- Go to managed Identities -> find the Agent Pool MI (agic-cluster-agentpool) -> Click on it
- Go to **Overview** -> Make a note of **Client ID**
- Update in **azure.json** value for **userAssignedIdentityID**
```t
# Get value for userAssignedIdentityID
  "userAssignedIdentityID": "b1a0abb5-6c7d-45cc-bbc8-5f89fe0aa50f"
```

## Step-07: Create azure.json file
- Update `resourceGroup` with value from `$AZURE_DNS_ZONE_RESOURCE_GROUP`. - In short, your DNS Zone resource group
```json
{
  "tenantId": "c81f465b-99f9-42d3-a169-8082d61c677a",
  "subscriptionId": "82808767-144c-4c66-a320-b30791668b0a",
  "resourceGroup": "dns-zones", 
  "useManagedIdentityExtension": true,
  "userAssignedIdentityID": "b1a0abb5-6c7d-45cc-bbc8-5f89fe0aa50f" 
}
```

## Step-08: Use the azure.json file to create a Kubernetes secret:
```t
# List k8s secrets
kubectl get secrets

# Create k8s secret with azure.json
cd 14-ExternalDNS-for-AzureDNS-on-AKS
kubectl create secret generic azure-config-file --namespace "default" --from-file azure.json

# List k8s secrets
kubectl get secrets

# k8s secret output as yaml
kubectl get secret azure-config-file -o yaml
Observation:
1. You should see a base64 encoded value
2. Decode it with https://www.base64decode.org/ to review it
```
 

## Step-09: Review external-dns.yaml manifest
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
rules:
  - apiGroups: [""]
    resources: ["services","endpoints","pods", "nodes"]
    verbs: ["get","watch","list"]
  - apiGroups: ["extensions","networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get","watch","list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
  - kind: ServiceAccount
    name: external-dns
    namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
        - name: external-dns
          image: registry.k8s.io/external-dns/external-dns:v0.13.5
          args:
            - --source=service
            - --source=ingress
            - --domain-filter=example.com # (optional) limit to only example.com domains; change to match the zone created above.
            - --provider=azure
            - --azure-resource-group=MyDnsResourceGroup # (optional) use the DNS zones from the tutorial's resource group
            - --txt-prefix=externaldns-
          volumeMounts:
            - name: azure-config-file
              mountPath: /etc/kubernetes
              readOnly: true
      volumes:
        - name: azure-config-file
          secret:
            secretName: azure-config-file
```

## Step-10: Deploy ExternalDNS
```t
# Deploy ExternalDNS 
kubectl apply -f kube-manifests/external-dns.yaml

# Verify ExternalDNS Logs
kubectl logs -f $(kubectl get po | egrep -o 'external-dns[A-Za-z0-9-]+')
```

```t
# Error Type: 400
time="2020-08-24T11:25:04Z" level=error msg="azure.BearerAuthorizer#WithAuthorization: Failed to refresh the Token for request to https://management.azure.com/subscriptions/82808767-144c-4c66-a320-b30791668b0a/resourceGroups/dns-zones/providers/Microsoft.Network/dnsZones?api-version=2018-05-01: StatusCode=400 -- Original Error: adal: Refresh request failed. Status Code = '400'. Response body: {\"error\":\"invalid_request\",\"error_description\":\"Identity not found\"}"

# Error Type: 403
Notes: Error 403 will come when our Managed Service Identity dont have access to respective destination resource 

# When all good, we should get log as below
time="2023-09-08T01:23:32Z" level=info msg="Instantiating new Kubernetes client"
time="2023-09-08T01:23:32Z" level=info msg="Using inCluster-config based on serviceaccount-token"
time="2023-09-08T01:23:32Z" level=info msg="Created Kubernetes client https://10.0.0.1:443"
time="2023-09-08T01:23:33Z" level=info msg="Using managed identity extension to retrieve access token for Azure API."
time="2023-09-08T01:23:33Z" level=info msg="Resolving to user assigned identity, client id is b1a0abb5-6c7d-45cc-bbc8-5f89fe0aa50f."
```

## External DNS References
- https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/azure.md
- https://github.com/kubernetes-sigs/external-dns
- https://github.com/kubernetes-sigs/external-dns/blob/master/docs/faq.md