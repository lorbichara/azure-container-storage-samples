# Introduction



## Getting Started with Azure Container Storage

This repo contains the code and instructions to deploy Azure Container Storage using CLI and deploy Jupyter & Kafka workloads.

### Pre-requisites (if not running on Cloud Shell)
* Install [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli#install-or-update)
* Install [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/#install-kubectl-binary-with-curl-on-windows)

### Installation

```bash
# Upgrade to the latest version of the aks-preview cli extension by running the following command.
az extension add --upgrade --name aks-preview

# Add or upgrade to the latest version of k8s-extension by running the following command.
az extension add --upgrade --name k8s-extension

# Set subscription context
az account set --subscription <subscription-id>

# Register resoure providers
az provider register --namespace Microsoft.ContainerService --wait 
az provider register --namespace Microsoft.KubernetesConfiguration --wait

# Create a resource group
az group create --name <resource-group-name> --location <location>

# Create an AKS cluster with Azure Container Storage extension enabled
az aks create -n <cluster-name> -g <resource-group-name> --node-vm-size Standard_D4s_v3 --node-count 3 --enable-azure-container-storage azureDisk

# Display available storage pools, one was created when Azure Container Storage was enabled
kubectl get sp –n acstor

# Display storage classes - there should be a storage class created that corresponds to the storage pool
kubectl get sc
```

## Jupyterhub

### Pre-requisites
* Install [Python](https://www.python.org/downloads/windows/) (if not running on Cloud Shell)
* Install [requests](https://pypi.org/project/requests/) Python library (if not running on Cloud Shell)
* Install [Chocolatey](https://chocolatey.org/install) to install helm (if not running on Cloud Shell)

### Deployment

1. Create config.yaml file to specify that each single user that will be created will get provisioned 1 Gi of storage, from a StorageClass created via Azure Container Storage.

```bash
code config.yaml
```
```bash
singleuser:
  storage:
    capacity: 1Gi
    dynamic:
      storageClass: acstor-azuredisk
hub:
  config:
    JupyterHub:
      admin_access: false
    Authenticator:
      admin_users:
        - admin
```
2. Install jupyterhub


```bash
(if not running on Cloud Shell) choco install kubernetes-helm
```

```bash
helm repo add jupyterhub https://jupyterhub.github.io/helm-chart/
helm repo update
```
This is installing JupyterHub using the information provided on config.yaml, and creating a namespace (jhub1) where the demo resources will sit.
```bash
helm upgrade --cleanup-on-fail --install jhub1 jupyterhub/jupyterhub --namespace jhub1 --create-namespace --values config.yaml
```
```bash
kubectl get service --namespace jhub1
```

3. Port forward to connect to the service
```bash
kubectl --namespace=jhub1 port-forward service/proxy-public 8080:http
```
Note: If the port-forward doesn't work from CloudShell, you can try using your computer's local terminal

#### Create additional user sessions
1. From the browser - log on to: http://localhost:8080/ using the credentials from the config.yaml file (username: admin).

2. Generate the token from http://localhost:8080/hub/token. Tokens are sent to the Hub for verification. The Hub replies with a JSON model describing the authenticated user.

3. Update the value of the token in the 'api_token' parameter in the Python script (user_creation.py)

4. Run python script
```bash
py user_creation.py
```

5. Run these commands in your cluster to get the pods and PVCs
```bash
kubectl get pvc -n jhub1
kubectl get pods -n jhub1
```

## ElasticSearch
### Cluster set up
1. Set environment variables
```bash
LOCATION=eastus # Location  

AKS_NAME=Ig23elsearch 

RG=IgniteACSDemo 

AKS_VNET_NAME=$AKS_NAME-vnet # The VNET where AKS will reside 

AKS_CLUSTER_NAME=$AKS_NAME # name of the cluster 

AKS_VNET_CIDR=172.16.0.0/16 

AKS_NODES_SUBNET_NAME=$AKS_NAME-subnet # the AKS nodes subnet 

AKS_NODES_NSG_NAME=$AKS_NAME-nsg # the AKS nodes subnet 

AKS_NODES_SUBNET_PREFIX=172.16.0.0/23 

SERVICE_CIDR=10.0.0.0/16 

DNS_IP=10.0.0.10 

NETWORK_PLUGIN=azure # use Azure CNI  

SYSTEM_NODE_COUNT=5  # system node pool size  

USER_NODE_COUNT=6 # change this to match your needs 

NODES_SKU=Standard_D4ds_v4 # node VM type (change this to match your needs) 

K8S_VERSION=1.27 

SYSTEM_POOL_NAME=systempool 

STORAGE_POOL_ZONE1_NAME=espoolz1 

IDENTITY_NAME=$AKS_NAME`date +"%d%m%y"` 
```
2. Create identity
```bash
az identity create --name $IDENTITY_NAME --resource-group $RG 
```
3. Get the identity ID and client ID - they'll be used later
```bash
IDENTITY_ID=$(az identity show --name $IDENTITY_NAME --resource-group $RG --query id -o tsv) 
IDENTITY_CLIENT_ID=$(az identity show --name $IDENTITY_NAME --resource-group $RG --query clientId -o tsv) 
```
4. Create a VNET and Subnet
```bash
az network vnet create \
  --name $AKS_VNET_NAME \
  --resource-group $RG \
  --location $LOCATION \
  --address-prefix $AKS_VNET_CIDR \
  --subnet-name $AKS_NODES_SUBNET_NAME \
  --subnet-prefix $AKS_NODES_SUBNET_PREFIX
```
5. Get the rg, VNET, and Subnet IDs
```bash
RG_ID=$(az group show -n $RG  --query id -o tsv) 
VNETID=$(az network vnet show -g $RG --name $AKS_VNET_NAME --query id -o tsv) 
AKS_VNET_SUBNET_ID=$(az network vnet subnet show --name $AKS_NODES_SUBNET_NAME -g $RG --vnet-name $AKS_VNET_NAME --query "id" -o tsv) 
```
6. Assign the managed identity permissions on the rg and VNET
```bash
az role assignment create --assignee $IDENTITY_CLIENT_ID --scope $RG_ID --role Contributor
az role assignment create --assignee $IDENTITY_CLIENT_ID --scope $VNETID --role Contributor
```
7. Validate role assignment
```bash
az role assignment list --assignee $IDENTITY_CLIENT_ID --all -o table
```
8. Create the cluster
```bash
az aks create --resource-group $RG --name $AKS_CLUSTER_NAME --location $LOCATION --os-sku AzureLinux --node-count $SYSTEM_NODE_COUNT --node-vm-size $NODES_SKU --network-plugin $NETWORK_PLUGIN --network-plugin-mode overlay --kubernetes-version $K8S_VERSION --generate-ssh-keys --service-cidr $SERVICE_CIDR --dns-service-ip $DNS_IP --vnet-subnet-id $AKS_VNET_SUBNET_ID --enable-addons monitoring --enable-managed-identity --assign-identity $IDENTITY_ID --nodepool-name $SYSTEM_POOL_NAME --uptime-sla --zones 1
```
9. Connect to the cluster
```bash
az aks get-credentials -n $AKS_CLUSTER_NAME -g $RG 
```
10. Check the nodes
```bash
kubectl get nodes -o wide 
```
11. Add a user nodepool
```bash
az aks nodepool add --cluster-name $AKS_CLUSTER_NAME --mode User --name $STORAGE_POOL_ZONE1_NAME --node-vm-size $NODES_SKU --resource-group $RG --zones 1 --enable-cluster-autoscaler --max-count 12 --min-count 6 --node-count $USER_NODE_COUNT --os-sku AzureLinux --labels app=es
```
12. Enable Azure Container Storage
```bash
az aks update --resource-group $RG --name $AKS_CLUSTER_NAME --enable-azure-container-storage azureDisk --azure-container-storage-nodepools $STORAGE_POOL_ZONE1_NAME 
```
13. Add nodepool label
```bash
az aks nodepool update --cluster-name Ig23elsearch --name systempool --resource-group $RG --labels acstor.azure.com/io-engine=acstor  
```

### Install Elastic Search
We'll use the "acstor-azuredisk" storage class, and use helm to install the ElasticSearch cluster. We'll rely on the ElasticSearch chart provided by bitnami.

1. Add the bitnami repository
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```
2. Get the values file we'll need to update
```shell
helm show values bitnami/elasticsearch > values_sample.yaml
```
We will create our own values file (there is a sample (values_acs.yaml) in this repo you can use) where we will:
1. Adjust the affinity and taints to match our node pools 
2. adjust the number of replicas and the scaling parameters for master, data, and coordinating, and ingestion nodes
3. configure the storage class 
4. optionally make the elastic search service accessible using a load balancer 
5. enable HPA for all the nodes
   
### Deploy Elastic Search
Now that we have configured the charts, we are ready to deploy the ES cluster
1. Create the namespace
```bash
kubectl create namespace elasticsearch
```
2. Install Elastic Search using the values file
```bash
helm install elasticsearch-v1 bitnami/elasticsearch -n elasticsearch --values values_acs.yaml
```
3. Validate the installation - it will take around 5 minutes for all the pods to move to a 'READY' state
```bash
watch kubectl get pods -o wide -n elasticsearch
```
4. Check the service so we can access Elastic Search - note the "External-IP"
```bash
kubectl get svc -n elasticsearch elasticsearch-v1
```
5. Port forward
```bash
kubectl -n elasticsearch port-forward svc/elasticsearch-v1 9200:9200 &
curl http://$SERVICE_IP:9200/
```
6. Create an index
```bash
##create an index called "acstor" with 3 replicas 
curl -X PUT "http://$esip:9200/acstor" -H "Content-Type: application/json" -d '{
  "settings": {
    "number_of_replicas": 3
  }
}'

##test the index 
curl -X GET "http://$esip:9200/acstor"
```
7. Ingest some data in Elastic Search using Python
8. Install Docker
9. Download the Dockerfile and ingest_logs.py and build the Docker image
10. Create an Azure Container Registry (ACR) from the portal
11. Change the registry name to match yours
```bash
#Point the folder to the one having docker file
az acr login --name IgniteCR
az acr update --name IgniteCR --anonymous-pull-enabled
docker build -t ignitecr.azurecr.io/my-ingest-image:1.0 .
docker push ignitecr.azurecr.io/my-ingest-image:1.0 
```

12. Run the job (remember to change the image name to yours in ingest-job.yaml) also change the parallelism and completions to match your needs
```bash
cd ..
kubectl apply -f ingest-job.yaml
```

13. Verify that the job is running
```bash
kubectl get pods -l app=log-ingestion 
kubectl logs -l app=log-ingestion -f 
```

14. Watch the Elastic Search pods being scaled out
```bash
watch kubectl get pods -n elasticsearch
kubectl get hpa -n elasticsearch 
```
