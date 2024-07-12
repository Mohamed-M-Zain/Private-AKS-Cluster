# Create AKS Cluster


## Step-01: Create Private cluster using Private Endpoint
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


## Step 03: Configure `kubectl` to Connect to Private AKS Cluster

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



