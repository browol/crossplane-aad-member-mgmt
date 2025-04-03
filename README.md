# Azure AD Group Member Management with Crossplane

Manage Azure AD groups and members using Crossplane. This guide provides a step-by-step process to set up Crossplane, install the Azure AD provider, and manage Azure AD groups and members.

## References

- [Crossplane Installation](https://docs.crossplane.io/v1.19/software/install/)
- [provider-azuread | Member](https://marketplace.upbound.io/providers/upbound/provider-azuread/v1.6.4/resources/groups.azuread.upbound.io/Member/v1beta1)
- [provider-azuread | Group](https://marketplace.upbound.io/providers/upbound/provider-azuread/v1.6.4/resources/groups.azuread.upbound.io/Group/v1beta2)

## Prerequisites

- Azure CLI installed and configured
- Kubernetes cluster running (e.g., using Kind)
- Helm installed
- K9s installed (optional)
- Kubectx and Kubens installed (optional)

## Create Kubernetes Cluster

```bash
kind create cluster
kubectx kind-kind
```

## Add and Update Crossplane Helm Repo

```bash
helm repo add \ 
crossplane-stable https://charts.crossplane.io/stable
helm repo update
```

## Install Crossplane

```bash
helm install crossplane \ 
crossplane-stable/crossplane \ 
--namespace crossplane-system \ 
--create-namespace
```

## Switch to Crossplane Namespace

```bash
kubens crossplane-system
```

## Verify Crossplane Installation

```bash
kubectl get pods -n crossplane-system
```

## Check Available Crossplane API Resources

```bash
kubectl api-resources | grep crossplane
```

## Install Azure AD Provider

```bash
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-azuread
spec:
  package: xpkg.upbound.io/upbound/provider-azuread:v1
EOF
```

## Verify Provider Installation

```bash
kubectl get provider
```

## Create Azure AD Service Principal and Credentials

```bash
az ad sp create-for-rbac --sdk-auth --name crossplane-azad > azuread-credentials.json
```

## Assign Required Permissions to Azure AD App

Ensure that the following permissions are assigned to the Azure AD App for the service principal:

| Permission Name | Type | Description | Admin Consent | Status |
|-----------------|------|-------------|---------------|--------|
| Group.ReadWrite.All | Application | Read and write all groups | Yes | Granted for Default Directory |
| GroupMember.ReadWrite.All | Application | Read and write all group memberships | Yes | Granted for Default Directory |
| User.Read.All | Application | Read all users' full profiles | Yes | Granted for Default Directory |

## Create Kubernetes Secret for Azure AD Credentials

```bash
kubectl create secret generic azuread-secret -n crossplane-system --from-file=creds=./azuread-credentials.json
```

## Configure Azure AD Provider

```bash
cat <<EOF | kubectl apply -f -
apiVersion: azuread.upbound.io/v1beta1
metadata:
  name: default
kind: ProviderConfig
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: azuread-secret
      key: creds
EOF
```

## Describe Provider Configurations

```bash
kubectl describe providerconfigs
```

## Create Azure AD Group

```bash
cat <<EOF | kubectl apply -f -
apiVersion: groups.azuread.upbound.io/v1beta2
kind: Group
metadata:
  annotations:
    meta.upbound.io/apiversion: groups/v1beta1/group
  labels:
    azadgroup.upbound.io/name: aks-reader
  name: aks-reader
spec:
  deletionPolicy: Orphan
  forProvider:
    displayName: aks-reader
    securityEnabled: true
  providerConfigRef:
    name: default
EOF
```

## Verify Managed Resources

```bash
kubectl get managed
```

## Describe Managed Resources

```bash
kubectl describe managed
```

## Add Member to Azure AD Group

```bash
cat <<EOF | kubectl apply -f -
apiVersion: groups.azuread.upbound.io/v1beta1
kind: Member
metadata:
  annotations:
    meta.upbound.io/apiversion: groups/v1beta1/member
    meta.upbound.io/objectid: xxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    meta.upbound.io/upn: yourname@example.com
  labels:
    azadmember.upbound.io/name: aks-reader-xxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
  name: aks-reader-xxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
spec:
  deletionPolicy: Delete
  forProvider:
    groupObjectIdSelector:
      matchLabels:
        azadgroup.upbound.io/name: aks-reader
    memberObjectId: xxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
  providerConfigRef:
    name: default
EOF
```

## Open K9s Dashboard for Crossplane Namespace

```bash
k9s --context kind-kind -n crossplane-system
```
