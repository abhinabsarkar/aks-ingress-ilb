# Enable TLS end-to-end (Application Gateway + Ingress) on AKS
When configured with end-to-end TLS communication mode, Application Gateway terminates the TLS sessions at the gateway and decrypts user traffic. It then applies the configured rules to select an appropriate backend pool instance to route traffic to. Application Gateway then initiates a new TLS connection to the backend server and re-encrypts data using the backend server's public key certificate before transmitting the request to the backend. Any response from the web server goes through the same process back to the end user. End-to-end TLS is enabled by setting protocol setting in Backend HTTP Setting to HTTPS, which is then applied to a backend pool. Refer [TLS termination and end to end TLS with Application Gateway](https://learn.microsoft.com/en-us/azure/application-gateway/ssl-overview)

TLS can be enabled on the ingress in the following ways:
1. Use Kubernetes secrets
2. Use AKV to store the certificates & use Secrets store CSI driver
3. Use TLS with Let's Encrypt certificate

```bash
export DOMAIN_NAME_AKS_BASELINE="abhinabsarkar.com"

# Create the certificate that will be presented to web clients by Azure Application Gateway for your domain
# The certificate subject name is appgwy.abhinabsarkar.com
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out frontend.crt -keyout frontend.key -subj "/CN=appgwy.${DOMAIN_NAME_AKS_BASELINE}/O=appgwy" -addext "subjectAltName = DNS:appgwy.${DOMAIN_NAME_AKS_BASELINE}" -addext "keyUsage = digitalSignature" -addext "extendedKeyUsage = serverAuth"

openssl pkcs12 -export -out frontend.pfx -in frontend.crt -inkey frontend.key -passout pass:password

# The certificate subject name is aks-ingress.abhinabsarkar.com
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out backend.crt -keyout backend.key -subj "/CN=aks-ingress.${DOMAIN_NAME_AKS_BASELINE}/O=AKS Ingress"

openssl pkcs12 -export -out backend.pfx -in backend.crt -inkey backend.key -passout pass:password
```

## Enable TLS using Kubernetes secrets
Install the certificate on the AKS cluster
```bash
kubectl create secret tls frontend-tls --key="frontend.key" --cert="frontend.crt"
kubectl create secret tls backend-tls --key="backend.key" --cert="backend.crt"

# validate the secrets getting added
kubectl get secrets
``` 

Deploy an application with HTTPS
```bash
kubectl apply -f https://raw.githubusercontent.com/Azure/application-gateway-kubernetes-ingress/master/docs/examples/sample-https-backend.yaml

# verify the application works internally in the AKS cluster
kubectl get pods

NAME                                  READY   STATUS    RESTARTS   AGE
website-deployment-579558bff8-77zpt   1/1     Running   0          42s
website-deployment-579558bff8-hw7f4   1/1     Running   0          42s

kubectl exec -it website-deployment-579558bff8-77zpt -- curl -k https://localhost:8443

Hello World!
```

Create ingress rule to connect to the service & test ingress controller
```bash
kubectl apply -f ingress-rule-tls.yaml

# validate ingress created
kubectl get ingress -n default

NAME              CLASS   HOSTS                           ADDRESS    PORTS     AGE
website-ingress   nginx   aks-ingress.abhinabsarkar.com   10.0.1.5   80, 443   32s

# look through the ingress settings
kubectl describe ingress website-ingress -n default
```
