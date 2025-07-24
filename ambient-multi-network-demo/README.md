# Testing Ambient Multi Network in AKS (istioctl)

Configuration instructions provided for multi-primary ambient mode multicluster.

Note: This demo is designed for use with Azure Kubernetes Service (AKS) clusters and requires the Azure CLI and kubectl to be installed.

## Setup

### Configure Clusters

```bash
# Step 1: Set variables
export RESOURCE_GROUP="myResourceGroup"
export CLUSTER_NAME="myAKSCluster"
export ACR_NAME="myContainerRegistry"
export LOCATION="eastus"
export CLUSTER_A_NAME="cluster-a"
export CLUSTER_B_NAME="cluster-b"

# Step 2: Create the resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Step 3: Create the ACR
az acr create --resource-group $RESOURCE_GROUP --name $ACR_NAME --sku Basic

# Step 4: Create the AKS clusters and attach ACR
az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_A_NAME \
  --node-count 3 \
  --enable-addons monitoring \
  --generate-ssh-keys \
  --attach-acr $ACR_NAME

az aks create \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_B_NAME \
  --node-count 3 \
  --enable-addons monitoring \
  --generate-ssh-keys \
  --attach-acr $ACR_NAME
```

### Setup Environment

```bash
# Step 5: Connect to the AKS clusters
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_A_NAME
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_B_NAME

# Step 6: Download the istioctl
container_id=$(docker create gcr.io/istio-testing/istioctl:1.27-dev)
docker cp $container_id:/usr/local/bin/istioctl .
# Step 7: Create istio-system namespace in each cluster
kubectl create namespace istio-system --context=$CLUSTER_A_NAME
kubectl create namespace istio-system --context=$CLUSTER_B_NAME
```

### Configure Root and Intermediate Certificates
Reference: https://istio.io/latest/docs/tasks/security/cert-management/plugin-ca-cert/
```bash
# Step 8: Create the root and intermediate certificates
wget https://raw.githubusercontent.com/istio/istio/release-1.21/tools/certs/common.mk -O common.mk
wget https://raw.githubusercontent.com/istio/istio/release-1.21/tools/certs/Makefile.selfsigned.mk -O Makefile.selfsigned.mk
# Step 9: Generate root CA
make -f Makefile.selfsigned.mk root-ca
# Step 10: Generate intermediate CAs
make -f Makefile.selfsigned.mk $CLUSTER_A_NAME-cacerts
make -f Makefile.selfsigned.mk $CLUSTER_B_NAME-cacerts
# Step 11: Create the cacert secret in each cluster
kubectl create secret generic cacerts -n istio-system \
      --from-file=$CLUSTER_A_NAME/ca-cert.pem \
      --from-file=$CLUSTER_A_NAME/ca-key.pem \
      --from-file=$CLUSTER_A_NAME/root-cert.pem \
      --from-file=$CLUSTER_A_NAME/cert-chain.pem

kubectl create secret generic cacerts -n istio-system \
      --from-file=$CLUSTER_B_NAME/ca-cert.pem \
      --from-file=$CLUSTER_B_NAME/ca-key.pem \
      --from-file=$CLUSTER_B_NAME/root-cert.pem \
      --from-file=$CLUSTER_B_NAME/cert-chain.pem
```

### Create Remote Secrets
Reference: https://istio.io/latest/docs/setup/install/multicluster/multi-primary/#enable-endpoint-discovery

```bash
# Step 12: Create remote secrets for each cluster
istioctl create-remote-secret \
    --context=$CLUSTER_A_NAME \
    --name=$CLUSTER_A_NAME | \
    kubectl apply -f - --context=$CLUSTER_B_NAME

istioctl create-remote-secret \
    --context=$CLUSTER_A_NAME \
    --name=$CLUSTER_A_NAME | \
    kubectl apply -f - --context=$CLUSTER_B_NAME
```

### Install Gateway API CRDs
```bash
# Step 13: Install Gateway API CRDs
kubectl get crd gateways.gateway.networking.k8s.io --context $CLUSTER_A_NAME &> /dev/null || \
  { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.3.0" --context $CLUSTER_A_NAME | kubectl apply --context $CLUSTER_A_NAME -f -; }

kubectl get crd gateways.gateway.networking.k8s.io --context $CLUSTER_B_NAME &> /dev/null || \
  { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.3.0" --context $CLUSTER_B_NAME | kubectl apply --context $CLUSTER_B_NAME -f -; }
```

## Demo

Demonstrate local, global, and waypoint routing with Istio Ambient Mesh in a multi-network setup. There are 3 variants of the helloworld service: local, global, and waypoint. The local service is only accessible within the cluster, the global service is accessible across clusters, and the waypoint service is accessible across clusters but requires routing through the L7 waypoint. Requests in this demo will be make from the curl service to one of the 3 variants of the helloworld service. Version 1 of the helloworld service will be deployed in cluster A and version 2 will be deployed in cluster B. This allows us to visualize the routing behavior across clusters.

### Setup Environment Variables
```bash
# Step 14: Set environment variables for cluster networks
export NETWORK_A_NAME=network1
export NETWORK_B_NAME=network2
```
### Install istio with istioctl

```bash
# Step 15: Install Istio in each cluster
istioctl install --context=$CLUSTER_A_NAME --manifest ./cluster-a-values.yaml
istioctl install --context=$CLUSTER_B_NAME --manifest ./cluster-b-values.yaml
```

### Deploy East West Gateways
```bash
# Step 16: Deploy East West Gateways
kubectl apply -f ./cluster-a-ewgateway.yaml --context=$CLUSTER_A_NAME
kubectl apply -f ./cluster-b-ewgateway.yaml --context=$CLUSTER_B_NAME
```

### Label istio-system namespace with cluster network name
```bash
# Step 17: Label the istio-system namespace in each cluster with the network name
kubectl --context="${CLUSTER_A_NAME}" get namespace istio-system && \
kubectl --context="${CLUSTER_A_NAME}" label namespace istio-system topology.istio.io/network=${NETWORK_A_NAME}

kubectl --context="${CLUSTER_B_NAME}" get namespace istio-system && \
kubectl --context="${CLUSTER_B_NAME}" label namespace istio-system topology.istio.io/network=${NETWORK_B_NAME}
```

### Deploy helloworld app
```bash
# Step 18: Deploy helloworld app in each cluster
kubectl apply -f ./cluster-a-helloworld.yaml --context=$CLUSTER_A_NAME
kubectl apply -f ./cluster-b-helloworld.yaml --context=$CLUSTER_B_NAME

### Deploy helloworld-local services
```bash
# Step 18: Deploy local services in each cluster
kubectl apply -f ./helloworld-local-service.yaml --context=$CLUSTER_A_NAME
kubectl apply -f ./helloworld-local-service.yaml --context=$CLUSTER_B_NAME
```

### Deploy helloworld-global services

Global services are visible across clusters from the istio data plane perspective. They have either a namespace or service label that indicates they are global.
```bash
# Step 19: Deploy global services in each cluster
kubectl apply -f ./helloworld-global-service.yaml --context=$CLUSTER_A_NAME
kubectl apply -f ./helloworld-global-service.yaml --context=$CLUSTER_B_NAME
```

### Deploy waypoint
Waypoints which correspond to backing global services must all be labeled global via service or namespace label.

```bash
# Step 20: Deploy waypoint services in each cluster
kubectl apply -f ./waypoint-for-helloworld.yaml --context=$CLUSTER_A_NAME
kubectl apply -f ./waypoint-for-helloworld.yaml --context=$CLUSTER_B_NAME
```

### Deploy helloworld-waypoint services
```bash
# Step 20: Deploy waypoint services in each cluster
kubectl apply -f ./helloworld-waypoint-service.yaml --context=$CLUSTER_A_NAME
kubectl apply -f ./helloworld-waypoint-service.yaml --context=$CLUSTER_B_NAME
```

### Deploy curl service
```bash
# Step 21: Deploy curl service in each cluster
kubectl apply -f ./curl.yaml --context=$CLUSTER_A_NAME
kubectl apply -f ./curl.yaml --context=$CLUSTER_B_NAME
```

### Enable ambient in the sample namespace
```bash
# Step 22: Enable ambient in the sample namespace
kubectl label namespace sample istio.io/dataplane-mode=ambient --context=$CLUSTER_A_NAME
kubectl label namespace sample istio.io/dataplane-mode=ambient --context=$CLUSTER_B_NAME
```

### Test the Setup

### Curl helloworld local

All request to the local helloworld service will be routed to the local service in the same cluster. The backing app is helloworld-v1.
```bash
# Step 23: Test local helloworld service
kubectl exec -it $(kubectl get pods -l app=curl -o jsonpath="{.items[0].metadata.name}" -n sample --context $CLUSTER_A_NAME) \
  -n sample --context $CLUSTER_A_NAME -- \
  /bin/sh -c 'for i in $(seq 1 10); do curl -s http://helloworld-local.sample.svc.cluster.local:5000/hello; done'
```

### Curl helloworld global
All requests to the global helloworld service will be routed to the global service on either cluster. The backing app is helloworld-v2 or helloworld-v1.
```bash
# Step 24: Test global helloworld service
kubectl exec -it $(kubectl get pods -l app=curl -o jsonpath="{.items[0].metadata.name}" -n sample --context $CLUSTER_A_NAME) \
  -n sample --context $CLUSTER_A_NAME -- \
  /bin/sh -c 'for i in $(seq 1 10); do curl -s http://helloworld-global.sample.svc.cluster.local:5000/hello; done'
```

### Curl helloworld waypoint
All requests to the waypoint helloworld service will be routed to the waypoint service on either cluster. By default the requests to services using waypoints will prefer local services if available. To force the request to go through the east-west gateway we have only deployed the global waypoint service on cluster B. The backing app is helloworld-v2 on cluster B.
```bash
# Step 25: Test waypoint helloworld service
kubectl exec -it $(kubectl get pods -l app=curl -o jsonpath="{.items[0].metadata.name}" -n sample --context $CLUSTER_A_NAME) \
  -n sample --context $CLUSTER_A_NAME -- \
  /bin/sh -c 'for i in $(seq 1 10); do curl -s http://helloworld-waypoint.sample.svc.cluster.local:5000/hello; done'
```