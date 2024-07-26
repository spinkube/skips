# SKIP 004 - Cloud Identity and Access Management

Summary: Many cloud Kubernetes services offer OIDC integration for authentication and authorization. This SKIP proposes to provide an opinionated way to manage IAM for SpinApps using workload identity and OIDC.

Owner: david@justice.dev (David Justice)

Impacted Projects:

- [x] spin-operator
- [ ] `spin kube` plugin
- [ ] runtime-class-manager
- [ ] containerd-shim-spin
- [ ] Governance
- [ ] Creates a new project

Created: July 25th, 2024

Updated: N/A

## Background

A Spin application is likely to use cloud services to peform tasks such as storing data, fetching runtime configuration variables, and other related tasks. These services often require IAM permissions to access. The current method of authentication is to load secrets into the runtime configuration for authentication to services. This is not ideal because it requires the developer to manage the secrets and the secrets are not rotated automatically.

## Proposal

Each SpinApp should be able to have a cloud identity associated with it granting access to the cloud resources it requires. This identity should be managed by the Spin operator and should be associated with the workload identity of the application. The workload identity should be associated with an OIDC provider, which can be used to authenticate to cloud services. The Spin operator should be able to reconcile the necessary cloud and cluster resources to enable this.

Multiple cloud providers support Kubernetes service account OIDC integration. For example:
- [Google Cloud](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)
- [AWS](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [Azure](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)

### SpinApp Resource Structure
#### SpinApp with Azure Workload Identity using Key Vault and CosmosDB
The yaml below shows an example of a SpinApp resource that uses Azure Workload Identity to authenticate to Azure Key Vault and CosmosDB. The `workloadIdentity` field is used to enable workload identity and specify the providers and service account to use. The `providers` field is used to specify the cloud providers that the workload identity will use. The `variableProviders` field is used to specify the variable providers that the workload identity will use. The `keyValueProviders` field is used to specify the key value providers that the workload identity will use. The `serviceAccount` field is used to specify the service account that the operator should create and associate to the workload.

```yaml
apiVersion: core.spinoperator.dev/v1alpha1
kind: SpinApp
metadata:
  name: wid-spinapp
spec:
  image: "ghcr.io/spinkube/containerd-shim-spin/examples/spin-rust-hello:v0.13.0"
  replicas: 1
  executor: containerd-shim-spin
  workloadIdentity:
    enabled: true
    providers:
      azure:
        tenantId: "9b9924f3-596a-4b18-a251-3de9eeef2bbc"
        variableProviders:
          - type: keyvault
            uri: "https://my-keyvault.vault.azure.net/"
        keyValueProviders:
          - type: cosmosdb
            account: "my-cosmosdb"
            database: "my-database"
            container: "my-container"
    serviceAccount: "my-service-account"
```

In this case the operator would be responsible for provisioning the following Azure resources, a user-assigned managed identity in Azure, an Azure Federated Identity Credential using the cluster's OIDC issurer URL, a CosmosDB SQL role assignment for read/write to the collection, and the Key Vault access policy to read secrets. Once those Azure resources have been created, then the operator should reconcile the service account and the SpinApp deployment.

The operator would then need to create the following Kubernetes resources (some details omitted for brevity):
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    "azure.workload.identity/client-id": "YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY" # The client ID of the user-assigned managed identity
  name: "my-service-account"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wid-spinapp
spec:
  template:
    metadata:
      labels:
        "azure.workload.identity/use": "true"
    spec:
      containers:
      - name: wid-spinapp
        image: "ghcr.io/spinkube/containerd-shim-spin/examples/spin-rust-hello:v0.13.0"
```

#### SpinApp with Azure Workload Identity using an Existing Identity
The yaml below shows an example of a SpinApp resource that uses Azure Workload Identity using a pre-existing identity. This is useful for when the operator is not able to create identities. For example, if a company does not allow the operator rights to create identities, and identities must be created via a security team.

In this case, the operator would not try to create the identity, but would instead use the existing identity and reconcele the service account and decorate the deployment with the necessary annotations and service account name.

```yaml
apiVersion: core.spinoperator.dev/v1alpha1
kind: SpinApp
metadata:
  name: wid-spinapp
spec:
  image: "ghcr.io/spinkube/containerd-shim-spin/examples/spin-rust-hello:v0.13.0"
  replicas: 1
  executor: containerd-shim-spin
  workloadIdentity:
    enabled: true
    providers:
      azure:
        tenantID: "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"
        clientID: "YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY" # The client ID of the user-assigned managed identity
    serviceAccount: "my-service-account"
```

The operator would then need to create the following resources (some details omitted for brevity):
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    "azure.workload.identity/client-id": "YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY"
  name: "my-service-account"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wid-spinapp
spec:
  template:
    metadata:
      labels:
        "azure.workload.identity/use": "true"
    spec:
      containers:
      - name: wid-spinapp
        image: "ghcr.io/spinkube/containerd-shim-spin/examples/spin-rust-hello:v0.13.0"
```

#### SpinApp with Google Cloud Workload Identity
TODO(DJ)

#### SpinApp with AWS Workload Identity
TODO(DJ)

## Alternatives Considered

I believe the alternative is to continue using the runtime configuration model where secrets can be stored in a secure vault. There will still need to be an initial secret to access the vault, which isn't ideal, but it is a common pattern.
