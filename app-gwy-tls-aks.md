# App Gwy for an AKS cluster with TLS

```bash
rgName=rg-abhi-aks
location=canadacentral
```

```bash
# create a public ip for Application Gateway
pipName=pip-appgwy-ingress
az network public-ip create -g $rgName -l $location -n $pipName --sku Standard
```

Application Gateway requires a `dedicated subnet`. Subnet Size /24 is recommended for App Gwy Subnet. Refer [Application Gateway infrastructure configuration](https://learn.microsoft.com/en-us/azure/application-gateway/configuration-infrastructure)
```bash
# create a subnet for app gwy. App Gwy requires a dedicated subnet
vnName=vn-abhi
snNameAppGwy=sn-appgwy
az network vnet subnet create --name $snNameAppGwy --address-prefixes 10.0.2.0/24 --resource-group $rgName --vnet-name $vnName

# create application gateway with public IP created above & backend connected to ingress controller having private IP
# Here the 10.0.1.5 is the IP Address of the internal load balancer created by Nginx ingress controller
appGwyName=appgwy-aks-ingress
az network application-gateway create \
    --name $appGwyName -g $rgName -l $location \
    --sku Standard_v2 --public-ip-address $pipName \
    --vnet-name $vnName --subnet $snNameAppGwy --priority 1000 \
    --servers 10.0.1.5
```

> The Application Gateway is created with a default rule (rule1). This rule binds the default listener (appGatewayHttpListener) with the default backend pool (appGatewayBackendPool) and the default backend HTTP settings (appGatewayBackendHttpSettings).

At this stage, browsing the request using the public IP address of the Application Gateway will return the application running on AKS.

```bash
# To reach app1 use the IP address or explicitly add the path for app1
curl http://4.205.41.105
curl http://4.205.41.105/hello-world-one
# To reach app2, add the path for app2 along with IP address
curl http://4.205.41.105/hello-world-two
```