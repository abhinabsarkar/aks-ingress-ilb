https://devopscube.com/configure-ingress-tls-kubernetes/

```bash
kubectl create ns dev

kubectl apply -f hello-app.yaml 

kubectl create secret tls hello-app-tls \
    --namespace dev \
    --key backend.key \
    --cert backend.crt

kubectl apply -f ingress.yaml

kubectl describe ingress hello-app-ingress -n dev

kubectl get ingress -n dev
NAME                CLASS   HOSTS                           ADDRESS    PORTS     AGE
hello-app-ingress   nginx   aks-ingress.abhinabsarkar.com   10.0.1.5   80, 443   37s

curl https://aks-ingress.abhinabsarkar.com -k

kubectl get svc -n dev
```

![alt txt](/images/appgwy-backend-setting.jpg)

![alt txt](/images/appgwy-listener.jpg)

![alt txt](/images/appgwy-healthprobe.jpg)

## Still getting error 502

Next steps: 
* https://learn.microsoft.com/en-us/azure/application-gateway/ssl-overview#end-to-end-tls-with-the-v2-sku
* https://learn.microsoft.com/en-us/azure/application-gateway/application-gateway-end-to-end-ssl-powershell
* https://github.com/MicrosoftDocs/azure-docs/issues/44425#issuecomment-564634256

Frontend certificate for App Gwy using the self signed root certificate
```bash
# key used to sign the certificate requests, anyone holding this can sign certificates on your behalf
openssl genrsa -des3 -out rootCA.key 4096

# Create the certificate that will be presented to web clients by Azure Application Gateway for your domain
export DOMAIN_NAME_AKS_BASELINE="abhinabsarkar.com"
openssl req -x509 -new -nodes -days 1024 -key rootCA.key -sha256 -out frontendrootCA.crt -subj "/CN=appgwy.${DOMAIN_NAME_AKS_BASELINE}/O=appgwy" -addext "subjectAltName = DNS:appgwy.${DOMAIN_NAME_AKS_BASELINE}" -addext "keyUsage = digitalSignature" -addext "extendedKeyUsage = serverAuth"

# output to pfx format which  can include arbitrary number of private keys with accompanying X.509 certificates and a certificate authority chain
openssl pkcs12 -export -out frontendrootCA.pfx -in frontendrootCA.crt -inkey rootCA.key -passout pass:password

# output certificate .crt in PEM format
# openssl pkcs12 -in frontendrootCA.pfx -out frontendrootCA.crt -nokeys -clcerts

# convert from PEM to DER (.cer)
openssl x509 -inform pem -in frontendrootCA.crt -outform der -out frontendrootCA.cer
```

Backend SSL certificate signed using the root certificate
```bash
# create the certificate key
openssl genrsa -out backend.key 2048
# Create the signing (csr)
openssl req -new -sha256 -key backend.key -subj "/CN=aks-ingress.${DOMAIN_NAME_AKS_BASELINE}/O=AKS Ingress" -out backend.csr
# verify the csr
openssl req -in backend.csr -noout -text
# Generate the certificate using the csr and key along with the CA Root key
openssl x509 -req -in backend.csr -CA frontendrootCA.crt -CAkey rootCA.key -CAcreateserial -out backend.crt -days 500 -sha256
# verify certificate's content
openssl x509 -in backend.crt -text -noout

# convert from PEM to DER (.cer)
openssl x509 -inform pem -in backend.crt -outform der -out backend.cer
```