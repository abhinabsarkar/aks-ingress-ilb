# AKS - Ingress controller with internal load balancer

This article describes ingress traffic for an AKS cluster with Nginx ingress controller using internal load balancer. The AKS cluster in this case uses [kubenet networking](https://learn.microsoft.com/en-us/azure/aks/configure-kubenet#overview-of-kubenet-networking-with-your-own-subnet). Few points to notice:
* Ingress exposes HTTP(s) routes from outside the cluster to services within the cluster
* Ingress controller creates & manages Azure internal load balancer to fulfill ingress. Refer [k8s ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/#what-is-ingress)
* A VM is created on another subnet to validate access from internal network outside the AKS cluster

The architecture will look as shown below.

## Simplified - Ingress controller with internal load balancer
![alt txt](/images/ingress-internal-loadbalancer.jpg)

## Detailed - Ingress controller with internal load balancer
Looking closer to see how ingress controller routes traffic to the application pods:
* Ingress allows to expose multiple services to be exposed using a single IP (see 10.0.1.5)
* Ingress rule maps domain name (`ingress.abs.com`) to the app service based on routing (allows domain as well as path based)
  * Load balancer works at Layer 4 whereas Ingress works at Layer 7
* [Service](https://github.com/abhinabsarkar/k8s-networking/blob/master/concepts/service-readme.md) then routes traffic to the application pods
* Service is discovered using [k8s DNS i.e. CoreDNS](https://github.com/abhinabsarkar/k8s-networking/blob/master/concepts/k8s-networking-readme.md#dns)

![alt txt](/images/ingress-internal-loadbalancer-detailed.jpg)

**AKS networking default address ranges**  
--service-cidr 10.0.0.0/16 --> 192.168.0.0/16 (customized for this installation)  
--dns-service-ip 10.0.0.10 --> 192.168.0.10   (customized for this installation)  
--pod-cidr 10.244.0.0/16   
--docker-bridge-address 172.17.0.1/16

* [Networking address ranges explained](https://learn.microsoft.com/en-us/azure/aks/configure-kubenet#create-an-aks-cluster-with-system-assigned-managed-identities)

## Pre-requisite
* A resource group
* A service principal
* Contributor role granted to the service principal on the resource group

```bash
rgName=rg-abhi-aks
spName=sp-aks-cicd
spClientId=
spSecret=
tenantId=
subscriptionId=
```

[Azure login](https://learn.microsoft.com/en-us/cli/azure/authenticate-azure-cli)
```bash
# login using service principal
read -sp "Azure password: " AZ_PASS && echo && az login --service-principal -u $spClientId -p $AZ_PASS --tenant $tenantId
```

### Create VNet & subnet
```bash
# Create a VNet with a default subnet. Default VNet address space - 10.0.0.0/16, Subnet address - 10.0.0.0/24
vnName=vn-abhi
snNameVm=sn-vm
az network vnet create --name $vnName --resource-group $rgName --subnet-name $snNameVm
# create subnet for AKS
snNameAks=sn-aks
az network vnet subnet create --name $snNameAks --address-prefixes 10.0.1.0/24 --resource-group $rgName --vnet-name $vnName
```

### Create an AKS cluster on an existing VNet
```bash
subnetId=$(az network vnet subnet show --resource-group $rgName --vnet-name $vnName --name $snNameAks --query id -o tsv)

# create a cluster with system assigned managed identity & rbac
# Since the VNet address space is 10.0.0.0/16, the default service CIDR in AKS needs to be set explicitly so that 
# it doesn't overlap with it
location=canadacentral
aksClusterName=aks-abhi
az aks create \
     --resource-group $rgName \
     --location $location \
     --name $aksClusterName \
     --enable-managed-identity --generate-ssh-keys \
     --enable-aad --aad-tenant-id $tenantId \
     --load-balancer-sku standard \
     --node-count 1 --node-vm-size Standard_D2S_v3 \
     --network-plugin kubenet \
     --vnet-subnet-id $subnetId \
     --service-cidr 192.168.0.0/16 --dns-service-ip 192.168.0.10

# Validate cluster installation - download credentials and configure the Kubernetes CLI
az aks get-credentials --resource-group $rgName --name $aksClusterName --admin
```

### Create a VM in the existing subnet sn-vm
```bash
userName=
password=
# Get the windows image
size=Standard_D4s_v3
publisher=MicrosoftWindowsDesktop
offer=Windows-11
sku=win11-21h2-pro
echo "Get the image OS.."
osImage=$(az vm image list --publisher $publisher --offer $offer --sku $sku --all --query "[0].urn" -o tsv)
# Create VM. Pass the parameters when running the shell script
# Parameters required: admin-username & admin-password
vmName=vm-abhi
location=canadacentral
az vm create \
    --resource-group $rgName --name $vmName \
    --image $osImage --size $size --location $location \
    --admin-username $userName --admin-password $vmPassword \
    --vnet-name $vnName --subnet $snNameVm \
    --tags Identifier=VM-Dev \
    --public-ip-address-allocation static --public-ip-sku Standard \
    --verbose

# Enable auto shutdown every night at 9:pm EST hours
echo "Enable auto shutdown every night at 9:pm EST hours.."
az vm auto-shutdown -g $rgName -n $vmName --time 0100
```

### Create an ingress controller with private IP 
* [Create an ingress controller in Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/ingress-basic?tabs=azure-cli#customized-configuration)

> In this case, I have created the VNet first & then created AKS cluster with system-assigned managed identity. Hence, the managed identity has to be granted network contributor access on the VNet. For ease of access, I have granted contributor access on the resource group itself.

```bash
# create a container registry. required for customization of ingress controller
az acr create -n acrabhi1 -g $rgName --sku Standard
# Associate the registry with AKS. This ensures that the managed identity has access to pull the images from the registry
az aks update -n $aksClusterName -g $rgName --attach-acr acrabhi1

REGISTRY_NAME=acrabhi1
SOURCE_REGISTRY=k8s.gcr.io
CONTROLLER_IMAGE=ingress-nginx/controller
CONTROLLER_TAG=v1.2.1
PATCH_IMAGE=ingress-nginx/kube-webhook-certgen
PATCH_TAG=v1.1.1
DEFAULTBACKEND_IMAGE=defaultbackend-amd64
DEFAULTBACKEND_TAG=1.5

az acr import --name $REGISTRY_NAME --source $SOURCE_REGISTRY/$CONTROLLER_IMAGE:$CONTROLLER_TAG --image $CONTROLLER_IMAGE:$CONTROLLER_TAG
az acr import --name $REGISTRY_NAME --source $SOURCE_REGISTRY/$PATCH_IMAGE:$PATCH_TAG --image $PATCH_IMAGE:$PATCH_TAG
az acr import --name $REGISTRY_NAME --source $SOURCE_REGISTRY/$DEFAULTBACKEND_IMAGE:$DEFAULTBACKEND_TAG --image $DEFAULTBACKEND_IMAGE:$DEFAULTBACKEND_TAG
```

create internal-ingress.yaml using `vi` editor
```yaml
controller:
  service:
    loadBalancerIP: 10.0.1.5 #IP Address from the VNet address space. Ensure the IP is available & not in use
    annotations:
      service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```

Deploy a customized ingress controller using an internal static IP address

> When you deploy the nginx-ingress chart with Helm, add the `-f internal-ingress.yaml` parameter
```bash
# Add the ingress-nginx repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Set variable for ACR location to use for pulling images
ACR_URL=acrabhi1.azurecr.io

# Use Helm to deploy an NGINX ingress controller
helm install ingress-nginx ingress-nginx/ingress-nginx \
    --version 4.1.3 \
    --namespace ingress-basic \
    --create-namespace \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.image.registry=$ACR_URL \
    --set controller.image.image=$CONTROLLER_IMAGE \
    --set controller.image.tag=$CONTROLLER_TAG \
    --set controller.image.digest="" \
    --set controller.admissionWebhooks.patch.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz \
    --set controller.admissionWebhooks.patch.image.registry=$ACR_URL \
    --set controller.admissionWebhooks.patch.image.image=$PATCH_IMAGE \
    --set controller.admissionWebhooks.patch.image.tag=$PATCH_TAG \
    --set controller.admissionWebhooks.patch.image.digest="" \
    --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
    --set defaultBackend.image.registry=$ACR_URL \
    --set defaultBackend.image.image=$DEFAULTBACKEND_IMAGE \
    --set defaultBackend.image.tag=$DEFAULTBACKEND_TAG \
    --set defaultBackend.image.digest="" \
    -f internal-ingress.yaml
```  

The above commands will create the following resources:
* azure internal loadbalancer with private ip 10.0.1.5
* nginx ingress controller
* namespace ingress-basic

```bash
kubectl get services --namespace ingress-basic -o wide -w ingress-nginx-controller

NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE     SELECTOR
ingress-nginx-controller   LoadBalancer   192.168.48.79   10.0.1.5      80:30164/TCP,443:30049/TCP   6m40s   app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx
```

## Test the configurations
* [Deploying an ingress controller to an internal virtual network and fronted by an Azure Application Gateway with WAF](https://gaunacode.com/deploying-an-ingress-controller-to-an-internal-virtual-network-and-fronted-by-an-azure-application-gateway-with-waf#heading-deploying-two-test-applications)
* [Create an ingress controller in Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/ingress-basic?tabs=azure-cli#run-demo-applications)
```bash
kubectl create ns ingress-test

kubectl apply -f test-app1.yaml
kubectl apply -f test-app2.yaml
```

Create ingress rule(s) to connect to the service & test ingress controller. This will create an ingress rule to map requests for that domain to the appropriate service in Kubernetes.
```bash
export DOMAIN_NAME="ingress.abs.com"
kubectl apply -f ingress-rule.yaml
```
> Note annotation of nginx.ingress.kubernetes.io/ssl-redirect: "false". This will ensure that if we ever assign a TLS certificate to the ingress definition, NGINX won't start re-routing http traffic to https. This is important for us since we are using an Application Gateway that will terminate SSL and we want this behavior to be enforced by the Application Gateway, not the ingress controller. Otherwise, it might mess up with the health probes from the Application Gateway to the Kubernetes cluster.

To test the application, login to the VM created in subnet `sn-vm`. Since the load balancer is using private IP address, the apps can be accessed only from the same network segment & not from internet/public IP address.
```bash
# Using the Host header to ensure our ingress route is used.

# To reach app1 use the hostname or explicitly add the path for app1
curl -L -H "Host: ingress.abs.com" http://10.0.1.5
curl -L -H "Host: ingress.abs.com" http://10.0.1.5/hello-world-one

# To reach app2, add the path for app2
curl -L -H "Host: ingress.abs.com" http://10.0.1.5/hello-world-two
```

To test it from browser, update the host file with the IP address of load balancer & domain name.

It can also be tested by creating a test pod and attaching a terminal session to it. Refer [test an internal IP address](https://learn.microsoft.com/en-us/azure/aks/ingress-basic?tabs=azure-cli#test-an-internal-ip-address)

Look through the objects created so far to understand the network configuration
```bash
# ingress details
kubectl describe ingress -n ingress-test
# service details
kubectl get services --all-namespaces
# app pod details
kubectl get pods -n ingress-test -o wide
```