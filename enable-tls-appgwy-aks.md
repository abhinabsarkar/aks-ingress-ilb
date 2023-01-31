# Enable TLS on Application Gateway (SSL offload & termination) for AKS

## Provision TLS self-signed certificates

* Refer [Generate Your Client-Facing and AKS Ingress Controller TLS Certificates](https://github.com/abhinabsarkar/aks-baseline/blob/main/02-ca-certificates.md)
    * For user-facing site, use an Extended Validation cert from CA. This will serve in front of the Azure Application Gateway. 
    * Domain Validation cert from CA to be used with the AKS Ingress Controller, as it will not be user facing.

Generate a client-facing self-signed TLS certificate

```bash
# variable for the domain name
export DOMAIN_NAME_AKS_BASELINE="abhinabsarkar.com"

# Create the certificate that will be presented to web clients by Azure Application Gateway for your domain
# The certificate subject name is appgwy.abhinabsarkar.com
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out appgw.crt -keyout appgw.key -subj "/CN=appgwy.${DOMAIN_NAME_AKS_BASELINE}/O=appgwy" -addext "subjectAltName = DNS:appgwy.${DOMAIN_NAME_AKS_BASELINE}" -addext "keyUsage = digitalSignature" -addext "extendedKeyUsage = serverAuth"

openssl pkcs12 -export -out appgw.pfx -in appgw.crt -inkey appgw.key -passout pass:password
```

Generate the wildcard certificate for the AKS Ingress Controller
```bash
# The wildcard certificate subject name is *.aks-ingress.abhinabsarkar.com
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out nginx-internal-aks-ingress.crt -keyout nginx-internal-aks-ingress.key -subj "/CN=*.aks-ingress.${DOMAIN_NAME_AKS_BASELINE}/O=AKS Ingress"
```

## Enable TLS on Application Gateway

Update the listener on Application Gateway to HTTPS protocol. Refer [Configure an Application Gateway with TLS termination](https://learn.microsoft.com/en-us/azure/application-gateway/create-ssl-portal#create-an-application-gateway)

```bash
# To reach app1 use the IP address or explicitly add the path for app1
curl https://4.204.187.193 -k
curl https://4.204.187.193/hello-world-one -k
# To reach app2, add the path for app2 along with IP address
curl https://4.204.187.193/hello-world-two -k
```

> The image in the response is not displayed. This is a separate issue from enabling TLS and hence skipped for now.

Add custom domain name to the IP address. 
* Refer [Add custom domain to public IP address](https://learn.microsoft.com/en-us/azure/virtual-machines/custom-domain#add-custom-domain-to-vm-public-ip-address)

```bash
# To reach app1 use the domain name or explicitly add the path for app1
curl https://abhinabsarkar.com -k
curl https://abhinabsarkar.com/hello-world-one -k
# To reach app2, add the path for app2 along with IP address
curl https://abhinabsarkar.com/hello-world-two -k 
```

The url from browser shows the certificate as `appgwy.abhinabsarkar.com`

## References
* [Generate Your Client-Facing and AKS Ingress Controller TLS Certificates](https://github.com/abhinabsarkar/aks-baseline/blob/main/02-ca-certificates.md)
* [Configure an Application Gateway with TLS termination](https://learn.microsoft.com/en-us/azure/application-gateway/create-ssl-portal#create-an-application-gateway)
* [Add custom domain to public IP address](https://learn.microsoft.com/en-us/azure/virtual-machines/custom-domain#add-custom-domain-to-vm-public-ip-address)