# AKS Identity Guide

## Azure AD Workload Identity & AAD Pod Identity

### Implementation and Migration Documentation

------------------------------------------------------------------------

# Overview

Azure Kubernetes Service (AKS) provides multiple mechanisms for
workloads to securely access Azure resources without embedding
credentials in code.

The two primary approaches are:

-   **Azure AD Workload Identity (Recommended)**
-   **AAD Pod Identity (Legacy)**

This document covers:

-   Initial implementation of both approaches\
-   Architecture and best practices\
-   Migration from AAD Pod Identity to Workload Identity

------------------------------------------------------------------------

# Part 1 --- Initial Implementations

------------------------------------------------------------------------

# 1. Azure AD Workload Identity (Recommended)

## Description

Azure AD Workload Identity enables Kubernetes workloads to authenticate
to Azure AD using **OIDC federation** and Kubernetes **Service
Accounts**. It removes the need for node-managed identities and proxy
components.

------------------------------------------------------------------------

## Prerequisites

-   AKS cluster\
-   Azure CLI with AKS support\
-   OIDC issuer enabled\
-   Permissions to create Managed Identities\
-   kubectl access

------------------------------------------------------------------------

## Implementation Steps

### Step 1 --- Enable OIDC and Workload Identity

``` bash
az aks update \
  --name <aks-name> \
  --resource-group <rg> \
  --enable-oidc-issuer \
  --enable-workload-identity
```

------------------------------------------------------------------------

### Step 2 --- Create Managed Identity

``` bash
az identity create \
  --name <identity-name> \
  --resource-group <rg>
```

Record:

-   Client ID\
-   Tenant ID\
-   Resource ID

------------------------------------------------------------------------

### Step 3 --- Create Federated Credential

``` bash
az identity federated-credential create \
  --name <fic-name> \
  --identity-name <identity-name> \
  --resource-group <rg> \
  --issuer <aks-oidc-issuer-url> \
  --subject system:serviceaccount:<namespace>:<sa-name>
```

------------------------------------------------------------------------

### Step 4 --- Create Kubernetes Service Account

``` yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <sa-name>
  namespace: <namespace>
  annotations:
    azure.workload.identity/client-id: "<managed-identity-client-id>"
```

------------------------------------------------------------------------

### Step 5 --- Update Pod Spec

``` yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: my-sa
```

------------------------------------------------------------------------

## Validation

-   Exec into pod\
-   Request token\
-   Access Azure resource (Key Vault, Storage, etc.)\
-   Confirm successful authentication

------------------------------------------------------------------------

## Best Practices

-   One identity per workload\
-   Apply least-privilege RBAC\
-   Separate identities per environment\
-   Monitor token usage and access logs

------------------------------------------------------------------------

# 2. AAD Pod Identity (Legacy)

> ⚠️ Not recommended for new deployments

------------------------------------------------------------------------

## Description

AAD Pod Identity enables pods to use Azure AD Managed Identities by
deploying components that intercept token requests to Azure IMDS.

------------------------------------------------------------------------

## Prerequisites

-   AKS cluster\
-   Managed Identity\
-   kubectl access\
-   Helm

------------------------------------------------------------------------

## Implementation Steps

### Step 1 --- Deploy Components

Deploy:

-   MIC (Managed Identity Controller)\
-   NMI (Node Managed Identity)

Typically installed via Helm.

------------------------------------------------------------------------

### Step 2 --- Create AzureIdentity

``` yaml
apiVersion: aadpodidentity.k8s.io/v1
kind: AzureIdentity
metadata:
  name: <identity-name>
spec:
  type: 0
  resourceID: <managed-identity-resource-id>
  clientID: <client-id>
```

------------------------------------------------------------------------

### Step 3 --- Create AzureIdentityBinding

``` yaml
apiVersion: aadpodidentity.k8s.io/v1
kind: AzureIdentityBinding
metadata:
  name: <binding-name>
spec:
  azureIdentity: <identity-name>
  selector: <selector>
```

------------------------------------------------------------------------

### Step 4 --- Label Pod

``` yaml
metadata:
  labels:
    aadpodidbinding: <selector>
```

------------------------------------------------------------------------

## Limitations

-   IMDS interception latency\
-   Additional components to manage\
-   Scaling limitations\
-   Maintenance mode (no future investment)

------------------------------------------------------------------------

# Part 2 --- Migration Guide

## AAD Pod Identity → Workload Identity

------------------------------------------------------------------------

# Migration Strategy

Migration should be:

-   Incremental\
-   Workload-by-workload\
-   Validated at each stage

Recommended approach:

-   Parallel run\
-   Gradual cutover\
-   Decommission legacy last

------------------------------------------------------------------------

## Step 1 --- Assess Current State

Inventory:

-   AzureIdentity resources\
-   AzureIdentityBindings\
-   Managed Identities\
-   Workloads using Pod Identity

------------------------------------------------------------------------

## Step 2 --- Enable Workload Identity

``` bash
az aks update \
  --enable-oidc-issuer \
  --enable-workload-identity
```

------------------------------------------------------------------------

## Step 3 --- Reuse or Create Managed Identities

-   Existing identities can be reused\
-   Reapply RBAC roles if needed

------------------------------------------------------------------------

## Step 4 --- Create Federated Credentials

Map identity to:

    system:serviceaccount:<namespace>:<service-account>

------------------------------------------------------------------------

## Step 5 --- Update Manifests

Replace:

    aadpodidbinding label

With:

-   Service Account reference\
-   Workload Identity annotation

------------------------------------------------------------------------

## Step 6 --- Testing

Validate:

-   Token retrieval\
-   Azure access\
-   Application logs\
-   Monitoring alerts

------------------------------------------------------------------------

## Step 7 --- Gradual Cutover

Migrate environments in order:

1.  Dev\
2.  Test\
3.  Production

------------------------------------------------------------------------

## Step 8 --- Decommission Pod Identity

After full migration:

-   Delete AzureIdentity resources\
-   Remove MIC/NMI deployments\
-   Remove CRDs

------------------------------------------------------------------------

# Migration Best Practices

-   Schedule maintenance windows\
-   Monitor authentication failures\
-   Maintain rollback manifests\
-   Document identity mappings

------------------------------------------------------------------------

# Comparison Summary

  Feature                    Workload Identity   AAD Pod Identity
  -------------------------- ------------------- -------------------
  Model                      OIDC federation     IMDS interception
  Extra Components           None                MIC + NMI
  Performance                High                Moderate
  Scalability                High                Limited
  Ops Overhead               Low                 Higher
  Kubernetes Alignment       Native              Indirect
  Microsoft Recommendation   ✅ Yes              ❌ No

------------------------------------------------------------------------

# Final Recommendation

Azure AD Workload Identity should be:

-   The **default for new AKS deployments**\
-   The **target state for existing clusters**

It provides:

-   Simpler architecture\
-   Improved security\
-   Better scalability\
-   Long-term Microsoft support

------------------------------------------------------------------------

**Author:** Mohsin Aleem\
**Last Updated:** 2026
