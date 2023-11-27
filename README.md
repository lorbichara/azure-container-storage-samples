# Introduction



## Getting Started with Azure Container Storage

This repo contains the code and instructions to deploy Azure Container Storage using CLI and deploy Jupyter & Kafka workloads.

### Pre-requisites
* Install [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli#install-or-update)
* Install [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/#install-kubectl-binary-with-curl-on-windows)

### Installation

```bash
# Upgrade to the latest version of the aks-preview cli extension by running the following command.
az extension add --upgrade --name aks-preview

# Add or upgrade to the latest version of k8s-extension by running the following command.
az extension add --upgrade --name k8s-extension

# set subscription context
az account set --subscription <subscription-id>

# register resoure providers
az provider register --namespace Microsoft.ContainerService --wait 
az provider register --namespace Microsoft.KubernetesConfiguration --wait

# create a resource group
az group create --name <resource-group-name> --location <location>

# create an AKS cluster with Azure Container Storage extension enabled
az aks create -n <cluster-name> -g <resource-group-name> --node-vm-size Standard_D4s_v3 --node-count 3 --enable-azure-container-storage <storage-pool-type>

# display available storage pools, one was created when Azure Container Storage was enabled
kubectl get sp –n acstor

# display storage classes - there should be a storage class created that corresponds to the storage pool
kubectl get sc
```

## Demo Jupytehub

### Pre-requisites
* Install [Python](https://www.python.org/downloads/windows/)
* Install [requests](https://pypi.org/project/requests/) Python library
* Install [Chocolatey](https://chocolatey.org/install) to install helm

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
choco install kubernetes-helm
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

## Demo Kafka
Kafka Message producer is a small Go lang application to generate X number of messages and ingest to Kafka. It's using confluent-kafka-go library. The kafka-x-messages-producer.yaml file requires some environment variables and the kafka user1 password as a secret. Environment variables are: NUM_MESSAGES, KAFKA_TOPIC, KAFKA_ADDR, KAFKA_USER and KAFKA_PASSWORD(as kubernetes secret).

```bash
export AZURE_SUBSCRIPTION_ID=<your_subscriptionID>
export AZURE_RESOURCE_GROUP="aks-kafka-san"
export AZURE_CLUSTER_NAME="aks-kafka-san"

# existing grafana and azure monitoring
grafanaId=$(az grafana show --name grafana-azure-ase --resource-group aks-azure --query id --output tsv)
azuremonitorId=$(az resource show --resource-group aks-azure --name grafana-azure-ase --resource-type "Microsoft.Monitor/accounts" --query id --output tsv)

az group create --name $AZURE_RESOURCE_GROUP --location australiaeast
az aks create -g $AZURE_RESOURCE_GROUP -n $AZURE_CLUSTER_NAME --generate-ssh-keys \
       --node-count 3 \
       --os-sku AzureLinux \
       --enable-azure-container-storage elasticSan \
       --node-vm-size standard_d8s_v5 \
       --max-pods=250 \
       --enable-azure-monitor-metrics \
       --azure-monitor-workspace-resource-id $azuremonitorId \
       --grafana-resource-id $grafanaId 
```

## Deploy Kafka from Bitnami using Helm

```bash
# Install Kafka using HELM
helm upgrade kafka --namespace kafka --create-namespace \
  --set controller.replicaCount=3 \
  --set global.storageClass=acstor-elasticSan \
  --set controller.heapOpts="-Xmx2048m -Xms2048m" \
  --set controller.persistence.enabled=True \
  --set controller.persistence.size=1Ti \
  --set controller.logPersistence.enabled=True \
  --set controller.logPersistence.size=100Gi \
  --set controller.resources.limits.cpu=2 \
  --set controller.resources.limits.memory=4Gi \
  --set controller.resources.requests.cpu=1 \
  --set controller.resources.requests.memory=2Gi \
  --set metrics.jmx.enabled=True \
  oci://registry-1.docker.io/bitnamicharts/kafka
```

## Create secret for default kafka user1 generated on the installation

```bash
kubectl create secret generic kafka-user-password \
--namespace=kafka \
--from-literal=password="$(kubectl get secret kafka-user-passwords --namespace kafka -o jsonpath='{.data.client-passwords}' | base64 -d | cut -d , -f 1)"
```

## Create Deployment to Produce X messages

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-x-messages-producer
  namespace: kafka
spec:
  replicas: 100
  selector:
    matchLabels:
      app: kafka-x-messages-producer
  template:
    metadata:
      labels:
        app: kafka-x-messages-producer
    spec:
      containers:
      - name: kafka-x-messages-producer
        image: docker.io/jorgearteiro/kafka-x-messages-producer:0.9.0
        command: ["./main"]
        env:
        - name: NUM_MESSAGES
          value: "10000000"
        - name: KAFKA_TOPIC
          value: "orders"
        - name: KAFKA_ADDR
          value: "kafka.kafka.svc.cluster.local:9092"
        - name: KAFKA_USER
          value: "user1"
        - name: KAFKA_PASSWORD
          valueFrom:
            secretKeyRef:
              name: kafka-user-password
              key: password
        resources:
          limits:
            cpu: "0.06"
            memory: "64Mi"
          requests:
            cpu: "0.01"
            memory: "32Mi"
```

## Creating Elastic San Pools using CRDs

```bash
kubectl apply -f - <<EOF
apiVersion: containerstorage.azure.com/v1alpha1
kind: StoragePool
metadata:
  name: san1
  namespace: acstor
spec:
  poolType:
    elasticSan: {}
  resources:
    requests: {"storage": 1Ti}
EOF
```

