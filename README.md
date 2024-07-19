# Deploy Private Azure Kubernetes Service Cluster with Azure Container Registry in a Landing Zone

## Overview

This guide provides instructions for deploying a private Azure Kubernetes Service (AKS) cluster and integrating it with Azure Container Registry (ACR) within a defined landing zone. Additionally, it covers deploying three applications using Kubernetes Ingress with SSL for secure access.


![AKS pdf and 55 more pages - Work - Microsoft​ Edge 7_19_2024 2_33_12 AM](https://github.com/user-attachments/assets/a6e3ec1f-25d6-43e6-8523-03d771c5de4b)


Step-01: Create Private cluster using Private Endpoint
- Create Kubernetes Cluster
- **Basics**
  - **Subscription:** choose the name of the subscription
  - **Resource Group:** Creat New: aks-RG1
  - **Cluster preset configuration:** Standard
  - **Kubernetes Cluster Name:** AKS_Cluster
  - **Region:** East US 
  - **Availability zones:** Zones 1
  - **AKS Pricing Tier:** Free
  - **Kubernetes Version:** Select what ever is latest stable version
  - **Automatic upgrade:** Enabled with patch
  - **Node Size:** Standard DS2 v2
  - **Scale method:** Autoscale
  - **Node Count range:** 1 to 5
- **Node Pools**
  - leave to defaults
- **Access**
  - **Authentication and Authorization:** 	Local accounts with Kubernetes RBAC
  - Rest all leave to defaults
- **Networking**
  - **Enable private cluster:** Check this box to allow access through a private endpoint.
  - **Network Configuration:** Azure CNI
  - **Traffic routing:** leave to defaults
  - **Security:** Leave to defaults
- **Integrations**
  - **Azure Container Registry:** None
  - All leave to defaults
- **Advanced**
  -  All leave to defaults
- **Tags**
  - leave to defaults
- **Review + Create**
  - Click on **Create**


## Step 02: Configure `kubectl` to Connect to Private AKS Cluster


<img width="561" alt="photo01" src="https://github.com/user-attachments/assets/fa4bc861-bea9-4adc-a37b-bb449ae6b556">

There are two ways to connect to a private AKS cluster:

### Method 1: Using DNS Resolver [Recommended] or DNS Forwarder Zone (On-premises)
1. **Create a DNS Resolver or DNS Forwarder Zone** in your on-premises DNS server.
   - Configure your on-premises DNS server to resolve the private DNS zone used by your AKS cluster.
   - Ensure that DNS queries for the AKS cluster are forwarded to the Azure private DNS zone.

### Method 2: Using a Jumpbox VM in Azure (Cloud)

1. **Create a Jumpbox VM** in another VNet and link it with the private DNS zone.

2. **Connect to the Jumpbox VM**.
   - SSH into the Jumpbox VM.
   - Install Azure CLI and `kubectl` on the Jumpbox VM if not already installed.

3. **Configure `kubectl` on the Jumpbox VM**.
   ```sh
   az aks get-credentials --resource-group aks-RG1 --name AKS_Cluster

   # List Kubernetes Worker Nodes
   kubectl get nodes 
   kubectl get nodes -o wide
   ```
  

## Step 03: create a Static Public IP for Ingress in Azure AKS

```t
# Get the resource group name of the AKS cluster 
az aks show --resource-group aks-RG --name AKS_Cluster --query nodeResourceGroup -o tsv

# TEMPLATE - Create a public IP address with the static allocation
az network public-ip create --resource-group <REPLACE-OUTPUT-RG-FROM-PREVIOUS-COMMAND> --name myAKSPublicIPForIngress --sku Standard --allocation-method static --query publicIp.ipAddress -o tsv

# REPLACE - Create Public IP: Replace Resource Group value
az network public-ip create --resource-group MC_aks-rg1_aksdemo1_centralus --name myAKSPublicIPForIngress --sku Standard --allocation-method static --query publicIp.ipAddress -o tsv
```
- Make a note of Static IP which we will use in next step when installing Ingress Controller


## Step 04: Install Ingress Controller

```t
# Install Helm3 (if not installed)
brew install helm

# Create a namespace for your ingress resources
kubectl create namespace ingress-basic

# Add the official stable repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

#  Customizing the Chart Before Installing. 
helm show values ingress-nginx/ingress-nginx

# Use Helm to deploy an NGINX ingress controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
    --namespace ingress-basic \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.service.externalTrafficPolicy=Local \
    --set controller.service.loadBalancerIP="REPLACE_STATIC_IP" 

# List Services with labels
kubectl get service -l app.kubernetes.io/name=ingress-nginx --namespace ingress-basic

# List Pods
kubectl get pods -n ingress-basic
kubectl get all -n ingress-basic
```

## Step 05: use ExternalDNS with AKS to automatically create DNS record sets in Azure DNS

- External-DNS needs permissions to Azure DNS to modify (Add, Update, Delete DNS Record Sets)
- We can provide permissions to External-DNS pod by Using Azure Managed Service Identity (MSI).



### Information Required for azure.json file
```t
# To get Azure Tenant ID
az account show --query "tenantId"

# To get Azure Subscription ID
az account show --query "id"
```

### Create azure.json file
```json
{
  "tenantId": "",
  "subscriptionId": "",
  "resourceGroup": "", 
  "useManagedIdentityExtension": true,
  "userAssignedIdentityID": ""  
}
```

## Step-06: Create MSI [Managed Service Identity] for External DNS to access Azure DNS Zones

### Create Manged Service Identity (MSI)
-  Managed Identities -> Add
- Resource Name: name of managed identities
- Resource group: name of resource group
- Location: choose location
- Click on **Create**

### Add Azure Role Assignment in MSI
- Opem MSI
- Click on **Azure Role Assignments** -> **Add role assignment**
- Scope: Resource group
- Resource group: name of resource group
- Role: Contributor

### Make a note of Client Id and update in azure.json
- Go to **Overview** -> Make a note of **Client ID"
- Update in **azure.json** value for **userAssignedIdentityID**

### Associate MSI in AKS Cluster VMSS
- Virtual Machine Scale Sets (VMSS) -> Open AKS_Cluster related VMSS
- Go to Settings -> Identity -> User assigned -> Add -> name of managed identities

### Create Kubernetes Secret and Deploy ExternalDNS
```t
# Create Secret
cd "ExternalDNS-for-AzureDNS-on-AKS\kube-manifests\ExternalDNS"
kubectl create secret generic azure-config-file --from-file=azure.json

# List Secrets
kubectl get secrets

# Deploy ExternalDNS 
cd "ExternalDNS-for-AzureDNS-on-AKS\kube-manifests\ExternalDNS"
kubectl apply -f external-dns.yml

# Verify ExternalDNS Logs
kubectl logs -f "name of pod"
```

## Step-07: Implement SSL using Lets Encrypt
 - To implement SSL using Let's Encrypt with your Azure Kubernetes Service (AKS) setup, you can use cert-manager. Cert-manager is a Kubernetes add-on that automates the management and issuance of TLS certificates from various issuing sources, including Let's Encrypt.

### Install Cert Manager
```t
# Label the ingress-basic namespace to disable resource validation
kubectl label namespace ingress-cert-basic cert-manager.io/disable-validation=true

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install cert-manager jetstack/cert-manager --namespace ingress--certbasic --version v1.13.3 --set installCRDs=true

 

# Verify Cert Manager pods
kubectl get pods --namespace ingress-cert-basic

# Verify Cert Manager Services
kubectl get svc --namespace ingress-cert-basic
```
### Deploy Cluster Issuer
```t
# Deploy Cluster Issuer
cd "CertManager-ClusterIssuer/kube-manifests"
kubectl apply -f cluster-issuer.yml

# List Cluster Issuer
kubectl get clusterissuer

# Describe Cluster Issuer
kubectl describe clusterissuer letsencrypt
```


## Step-08: Deploy k8s Applications Manifests

- 01-NginxApp1-Manifests
- 02-NginxApp2-Manifests
- 03-UserMgmtmWebApp-Manifests
- 04-IngressService[ssl]-Manifests

```t
# Deploy
cd "deploy apps with ssl\kube-manifests"
kubectl apply -R -f kube-manifests/

# Verify Pods
kubectl get pods

# Verify Cert Manager Pod Logs
kubectl get pods -n ingress-cert-basic
kubectl -n ingress-basic logs -f <cert-manager-55d65894c7-sx62f> -n ingress-cert-basic #Replace Pod name

# Verify SSL Certificates (It should turn to True)
``log
$ kubectl get certificate
NAME                   READY      SECRET              AGE
app1-inovasys-secret   True    app1-inovasys-secret   45m
app2-inovasys-secret   True    app2-inovasys-secret   45m
```

#Verify Record set got automatically deleted in DNS Zones
go to Azure DNS Zone and check if records are created automatically.


<img width="713" alt="photo02" src="https://github.com/user-attachments/assets/46559de8-2228-4983-811b-3c7ac3a177ef">

### Access Applications

### Access App1
https://app1.inovasys-lab.com/app1

### Access App2
https://app2.inovasys-lab.com/app2

### Access Usermgmt Web App
http://app3.inovasys-lab.com


## Step-09: Azure Container Registry ACR with AKS

- ACR is a managed Docker container registry service for storing and managing container images.
- there are the two ways to connect Azure Container Registry (ACR) with Azure Kubernetes Service (AKS):
  - Attach ACR Directly to AKS
    - This method simplifies the process by directly attaching the ACR to the AKS cluster. This way, the AKS cluster can pull images from ACR without needing additional configuration.

  - Use a Service Principal
    - If you don't directly attach ACR to AKS, you can use a service principal to grant the AKS cluster access to ACR.  

##  Step-10: using azure devops pipeline to build and push images to ACR

   - **Prerequisites**
     - Azure DevOps account
     - A Git repository connected to Azure DevOps
     - ACR and AKS set up as described in the previous steps

##  Step-11: using azure devops pipeline to deploy minafasts files to AKS by using images in ACR 





<img width="614" alt="Screenshot 2024-07-19 015333" src="https://github.com/user-attachments/assets/00714ce8-1f0c-474f-9933-232b7487f444">



<img width="614" alt="Screenshot 2024-07-19 015333" src="https://github.com/user-attachments/assets/23483cb2-c2f6-4360-a0b4-838d8cc24120">
