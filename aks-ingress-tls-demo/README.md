# AKS Ingress TLS Sample

This sample implements enabling TLS on Ingress endpoint of AKS cluster by syncing TLS certificates from Azure Key Vault using Secret Store CSI Driver - Azure Provider.

## Prerequisite

* Deploy or Upgrade AKS cluster to 1.16.8 K8s version

* Install nginx ingress controller

Download and install NGINX ingress helm chart

```
# using Helm3.x

helm repo add nginx-stable https://helm.nginx.com/stable

helm repo update

kubectl create ns ingress

helm install my-release nginx-stable/nginx-ingress
```

### Generate self-signed SSL certificate (Testing purpose only)
```
KEY_PATH="aks-ingress-tls.key"
CERT_PATH="aks-ingress-tls.crt"
CERT_WITH_KEY_PATH="aks-ingress-tls-withkey.pem"
HOST_NAME="<Your Host>"

openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout $KEY_PATH -out $CERT_PATH -subj '//CN=$HOST_NAME//O=$HOST_NAME'

cat $CERT_PATH $KEY_PATH > $CERT_WITH_KEY_PATH
```

### Add pem file as a certificate to Application Key Vault

```
az keyvault certificate import --vault-name <Your Azure Key Vault> -n cert_name -f cert_file
```

### Set access policy to Application Key Vault for aks runtime service principal

```sh
az keyvault set-policy -n <application-keyvault-name> --spn <aks-runtime-service-principal-applicationi-id> --secret-permissions get --key-permissions get --certificate-permissions get
```

## Secret Store CSI Driver

### Generate kubeconfig to connect AKS cluster

Retrieve AKS cluster kube config

```sh
az aks get-credentials --name <Your AKS Cluster> --resource-group <Your AKS Resource Group> --admin
```

Test AKS cluster connection

```sh
kubectl get all -A
```

### Install Secret store CSI driver and Azure KeyVault provider

[Azure Key Vault provider for Secrets Store CSI driver](https://github.com/Azure/secrets-store-csi-driver-provider-azure) allows you to get secret contents stored in an Azure Key Vault instance and use the Secrets Store CSI driver interface to mount them into Kubernetes pods.

```sh
kubectl create ns csi-secrets-store

helm repo add csi-secrets-store-provider-azure https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/master/charts

helm install csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --generate-name -n csi-secrets-store
```

Check whether Secrets Store CSI driver resources are created and running fine.

```sh
kubectl get all -n csi-secret-store
```

### Create K8s secret with the App ID of runtime service principal and Client Secret from AKS Key Vault

```sh
AKS_VAULT_NAME="<Your Azure Key Vault>"
CLIENT_ID=$(az keyvault secret show --name runtimeServicePrincipalClientID --vault-name $AKS_VAULT_NAME --query value)
CLIENT_SECRET=$(az keyvault secret show --name runtimeServicePrincipalSecret --vault-name $AKS_VAULT_NAME --query value)

kubectl create ns aks-ingress-tls

kubectl create secret generic secrets-store-creds --from-literal clientid=$CLIENT_ID --from-literal clientsecret=$CLIENT_SECRET -n aks-ingress-tls
```

### Update SecretProviderClass with secret store configuration

Update [this sample deployment](aks-ingress-tls-sample.yaml) to configure `SecretProviderClass` resource to provide Azure-specific parameters for the Secrets Store CSI driver.

```yml
apiVersion: secrets-store.csi.x-k8s.io/v1alpha1
kind: SecretProviderClass
metadata:
  name: aks-ingress-tls-demo
spec:
  provider: azure                                       # accepted provider options: azure or vault
  secretObjects:                                        # [OPTIONAL] SecretObject defines the desired state of synced K8s secret objects
  - secretName: aks-ingress-tls                         # name of the K8s Secret object
    type: kubernetes.io/tls                             # type of the K8s Secret object e.g. Opaque, kubernetes.io/tls
    data: 
    - objectName: aks-ingress-tls-crt                   # name of the mounted content to sync with K8s Secret object
      key: tls.key
    - objectName: aks-ingress-tls-crt
      key: tls.crt
  parameters:
    usePodIdentity: "false"                             # [OPTIONAL for Azure] if not provided, will default to "false"       
    useVMManagedIdentity: "false"                       # [OPTIONAL for Azure] if not provided, will default to "false"
    keyvaultName: "<Your Azure Key Vault>"              # name of the Azure KeyVault                   
    cloudName: ""                                       # [OPTIONAL] if not provided, azure environment will default to AzurePublicCloud
    objects:  |
      array:
        - |
          objectName: aks-ingress-tls-crt               # name of the secret in Azure KeyVault
          objectType: secret                            # object types: secret, key or cert
    tenantId: "<Your Azure AD Tenant>"                  # tenant ID of the Azure KeyVault
```

### Provide Identity to Access Key Vault

The Azure Key Vault Provider offers four modes for accessing a Key Vault instance. In this sample, we will be using Service Principal.

1. [Service Principal](https://github.com/Azure/secrets-store-csi-driver-provider-azure/blob/master/docs/service-principal-mode.md)
2. [Pod Identity](https://github.com/Azure/secrets-store-csi-driver-provider-azure/blob/master/docs/pod-identity-mode.md)
3. [VMSS User Assigned Managed Identity](https://github.com/Azure/secrets-store-csi-driver-provider-azure/blob/master/docs/user-assigned-msi-mode.md)
4. [VMSS System Assigned Managed Identity](https://github.com/Azure/secrets-store-csi-driver-provider-azure/blob/master/docs/system-assigned-msi-mode.md)

### Update Deployment yaml 

To ensure your application is using the Secrets Store CSI driver, update your deployment yaml to use the `secrets-store.csi.k8s.io` driver and reference the `SecretProviderClass` resource created in the previous step.

```yml
    spec:
      containers:
      - name: aks-ingress-tls-demo
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 200m
            memory: 128Mi
          requests:
            cpu: 100m
            memory: 128Mi
        volumeMounts:
        - name: aks-ingress-tls-demo
          mountPath: "/mnt/secrets-store"               # Path of secret volume mount
          readOnly: true
      volumes:
      - name: aks-ingress-tls-demo
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: aks-ingress-tls-demo   # Provide reference to Secret Provider Class
          nodePublishSecretRef:
            name: secrets-store-creds                   # Secret containing Service Principal Client ID and Client Secret
```

### Deploy Kubernetes resources

Apply aks-ingress-tls-sample.yaml to k8s
```sh
kubectl apply -f aks-ingress-tls-sample.yaml -n aks-ingress-tls
```

Check whether pods are running
```sh
kubectl get pods -n aks-ingress-tls
```

### Validate the secret
To validate, once the pod is started, you should see the new mounted content at the volume path specified in your deployment yaml.

Check whether secrets-store volume is mounted to the pods
```sh
## show secrets held in secrets-store
kubectl exec -it <pod-name> -n aks-ingress-tls -- ls /mnt/secrets-store/

## print secret held in secrets-store
kubectl exec -it <pod-name> -n aks-ingress-tls -- cat /mnt/secrets-store/aks-ingress-tls-crt
```

Check whether certificate is synced with K8s secret
```sh
kubectl get secret aks-ingress-tls -n aks-ingress-tls -o yaml
```

### Test ingress with TLS

```sh
curl -v -k --resolve <Your Host>:443:<Ingress Load Balancer External IP> https://<Your Host>
```

**For browser access, register ingress load balancer external-ip in private DNS zone**
