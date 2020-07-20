# Kubernetes-Secrets-Store-CSI-Driver

[Secrets Store CSI driver](https://github.com/kubernetes-sigs/secrets-store-csi-driver) - Integrates secrets stored in external vaults with Kubernetes.

The Secrets Store CSI driver secrets-store.csi.k8s.io allows Kubernetes to mount multiple secrets, keys, and certs stored in enterprise-grade external secrets stores into their pods as a volume. Once the Volume is attached, the data in it is mounted into the container's file system.

# How It Works

The diagram below illustrates how Secrets Store CSI Volume works.

![Kubernetes Secret Store CSI](./img/k8s-secretstore-csi.png)

# Providers

Provider defines the actions for Secrets Store CSI driver. This enables retrieval of sensitive objects stored in an enterprise-grade external secrets store into Kubernetes while continue to manage these objects outside of Kubernetes. 

Following providers are currently supported:

[Azure Provider](https://github.com/Azure/secrets-store-csi-driver-provider-azure) for Azure Key Vault

[Vault Provider](https://github.com/hashicorp/secrets-store-csi-driver-provider-vault) for Hashicorp Vault (EVA)

# Scenarios

## Retrieve TLS certificate for Ingress from Azure Key Vault

Refer [AKS Ingress TLS](/aks-ingress-tls/README.md) sample
