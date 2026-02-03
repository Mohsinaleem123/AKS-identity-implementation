# Integrate Azure Key Vault with AKS using Terraform (Step‑by‑Step)

This guide assumes:
- You are **completely new** to Azure + Terraform
- You are working **only in Azure Cloud Shell**
- You want **secure secret management** using **Azure Key Vault + AKS**
- Secrets will be **mounted into pods** (best practice)

We will use:
- Azure Kubernetes Service (AKS)
- Azure Key Vault
- Managed Identity
- Secrets Store CSI Driver
- Terraform

---

## 0. Architecture (What we are building)

```
Azure Key Vault (secrets)
        ↑
Managed Identity (AKS)
        ↑
AKS Cluster
        ↑
Kubernetes Pod (reads secrets securely)
```

No secrets stored in:
- Terraform state
- Kubernetes YAML
- GitHub

---

## 1. Open Azure Cloud Shell

1. Go to https://portal.azure.com
2. Click **Cloud Shell** (top‑right)
3. Choose **Bash**

Verify login:
```bash
az account show
```

---

## 2. Install Terraform in Cloud Shell

Terraform is **not always preinstalled**.

```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install terraform -y
```

Verify:
```bash
terraform version
```

---

## 3. Create Project Folder

```bash
mkdir aks-keyvault-terraform
cd aks-keyvault-terraform
```

---

## 4. Create Terraform Provider Config

Create `providers.tf`

```hcl
terraform {
  required_version = ">= 1.3"

  backend "azurerm" {
    resource_group_name  = "rg-terraform-state"
    storage_account_name = "tfstate11369"
    container_name       = "tfstate"
    key                  = "aks-keyvault.tfstate"
  }

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

```

---

## 5. Create Resource Group

Create `resource-group.tf`

```hcl
resource "azurerm_resource_group" "rg" {
  name     = "rg-aks-keyvault"
  location = "East US"
}
```

---

## 6. Create Azure Key Vault

Create `keyvault.tf`

```hcl
data "azurerm_client_config" "current" {}

resource "azurerm_key_vault" "kv" {
  name                       = "kv-aks-demo-${random_integer.suffix.result}"
  location                   = azurerm_resource_group.rg.location
  resource_group_name        = azurerm_resource_group.rg.name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "standard"

  purge_protection_enabled   = false
}

resource "random_integer" "suffix" {
  min = 10000
  max = 99999
}
```

---

## 7. Add a Secret to Key Vault

Create `keyvault-secret.tf`

```hcl
resource "azurerm_key_vault_access_policy" "admin" {
  key_vault_id = azurerm_key_vault.kv.id
  tenant_id    = data.azurerm_client_config.current.tenant_id
  object_id    = data.azurerm_client_config.current.object_id

  secret_permissions = ["Get", "Set", "List"]
}

resource "azurerm_key_vault_secret" "db_password" {
  name         = "db-password"
  value        = "SuperSecretPassword123"
  key_vault_id = azurerm_key_vault.kv.id
  depends_on = [
    azurerm_key_vault_access_policy.admin
  ]
}
```

---

## 8. Create AKS Cluster (with Managed Identity)

Create `aks.tf`

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "aks-keyvault-demo"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = "aks-kv-demo"

  default_node_pool {
    name       = "system"
    node_count = 2
    vm_size    = "Standard_DS2_v2"
  }

  identity {
    type = "SystemAssigned"
  }
}
```

Create `aks-workload.tf`

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "aks-keyvault-demo"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = "aks-kv-demo"

  workload_identity_enabled = true
  oidc_issuer_enabled       = true

  default_node_pool {
    name       = "system"
    node_count = 2
    vm_size    = "Standard_DS2_v2"
  }

  identity {
    type = "SystemAssigned"
  }
}

```
---

## 9. Give AKS Access to Key Vault

Create `aks-kv-permission.tf`

```hcl
data "azurerm_kubernetes_cluster" "aks" {
  name                = "aks-keyvault-demo"
  resource_group_name = "rg-aks-keyvault"
}

data "azurerm_kubernetes_cluster_node_pool" "agentpool" {
  name                = "agentpool"
  kubernetes_cluster_id = data.azurerm_kubernetes_cluster.aks.id
}

resource "azurerm_key_vault_access_policy" "aks" {
  key_vault_id = azurerm_key_vault.kv.id
  tenant_id    = data.azurerm_client_config.current.tenant_id
  object_id    = azurerm_kubernetes_cluster.aks.identity[0].principal_id

  secret_permissions = ["Get", "List"]
}
```

---

## 10. Deploy Infrastructure

```bash
terraform init
terraform plan
terraform apply
```

Type:
```bash
yes
```

⏳ AKS takes ~5–10 minutes

---

## 11. Connect kubectl to AKS

```bash
az aks get-credentials \
  --resource-group rg-aks-keyvault \
  --name aks-keyvault-demo
```

Verify:
```bash
kubectl get nodes
```

---

## 12. Install Secrets Store CSI Driver

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/secrets-store-csi-driver/main/deploy/rbac-secretproviderclass.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/secrets-store-csi-driver/main/deploy/csidriver.yaml
```

Install Azure provider:
```bash
kubectl apply -f https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/main/deployment/provider-azure-installer.yaml
```

```bash
az aks enable-addons --addons azure-keyvault-secrets-provider

kubectl get pods -n kube-system -l app=secrets-store-csi-driver
kubectl get pods -n kube-system -l app=secrets-store-provider-azure

# helm repo add csi-secrets-store-provider-azure \
#   https://azure.github.io/secrets-store-csi-driver-provider-azure/charts
# helm repo update

# helm repo add secrets-store-csi-driver \
#   https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
# helm repo update

# helm install csi-secrets-store \
#   secrets-store-csi-driver/secrets-store-csi-driver \
#   --namespace kube-system

# helm install csi-secrets-store-provider-azure \
#   csi-secrets-store-provider-azure/csi-secrets-store-provider-azure \
#   --namespace kube-system


# helm install csi-secrets-store \
#   secrets-store-csi-driver/secrets-store-csi-driver \
#   --namespace kube-system \
#   --set syncSecret.enabled=true

# helm repo add csi-secrets-store-provider-azure \
#   https://azure.github.io/secrets-store-csi-driver-provider-azure/charts
# helm repo update

# helm install csi-secrets-store-provider-azure \
#   csi-secrets-store-provider-azure/csi-secrets-store-provider-azure \
#   --namespace kube-system

# helm install csi-secrets-store-provider-azure csi-secrets-store-provider-azure/csi-secrets-store-provider-azure \
#   --namespace kube-system \
#   --set secrets-store-csi-driver.install=false


# helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts

# helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system


# leavel below

# kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/secrets-store-csi-driver/main/deploy/csidriver.yaml

# kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/secrets-store-csi-driver/main/deploy/rbac-secretproviderclass.yaml

# kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/secrets-store-csi-driver/main/deploy/csidriver.yaml
# kubectl apply -f https://raw.githubusercontent.com/Azure/secrets-store-csi-driver-provider-azure/main/deployment/provider-azure-installer.yaml



```


## 13. Create SecretProviderClass

Create `secret-provider.yaml`

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    keyvaultName: <YOUR_KEYVAULT_NAME>
    tenantId: <YOUR_TENANT_ID>
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
```

Apply:
```bash
kubectl apply -f secret-provider.yaml
```

---

## 14. Deploy Test Pod

Create `pod.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kv-test
spec:
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: secrets-store
      mountPath: "/mnt/secrets"
      readOnly: true
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: azure-keyvault
```

Apply:
```bash
kubectl apply -f pod.yaml
```

---

## 15. Verify Secret Access

```bash
kubectl exec -it kv-test -- cat /mnt/secrets/db-password
```

✅ You should see:
```
SuperSecretPassword123
```

---

---
