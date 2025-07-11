# Krateo Composition for Azure DevOps

This is a Krateo Composition that orchestrates the creation of a complete Azure DevOps project, including a Team Project, Environment, Git Repository, Pipeline, and Pipeline Permissions. 
It demonstrates the seamless integration of resources managed by both the Azure DevOps Provider (classic) and the Azure DevOps Provider KOG.

## Features

This Composition implements the following steps:
1.  **Creates an Azure DevOps Team Project**: A new Team Project is provisioned in your specified Azure DevOps organization, managed by the Azure DevOps Provider (classic).

---
TODO
2.  **Creates an Azure DevOps Environment**: An Environment is created within the new Team Project, managed by the Azure DevOps Provider (classic).
3.  **Creates an Azure DevOps Git Repository**: 
4.  **Creates an Azure DevOps Pipeline**: 
5.  **Grants Pipeline Permissions**: Permissions are granted for the created Pipeline to access the created Environment. This crucial step leverages the Azure DevOps Provider KOG, dynamically looking up the IDs of the Environment and Pipeline created by the classic provider.

## Requirements

Before installing this Composition, ensure you have the following:
- **Krateo PlatformOps** installed in your Kubernetes cluster.

### Setup toolchain on krateo-system namespace

```sh
helm repo add krateo https://charts.krateo.io
helm repo update krateo
helm install azuredevops-provider krateo/azuredevops-provider --namespace krateo-system --create-namespace
helm install azuredevops-provider-kog krateo/azuredevops-provider-kog --namespace krateo-system
```

### Create an Azure DevOps Personal Access Token (PAT) and Kubernetes Secret

You will need a Personal Access Token (PAT) with sufficient permissions to create and manage Azure DevOps resources.
You can create a PAT by following these steps:
1. Go to your Azure DevOps organization on the Azure DevOps portal.
2. On the top right corner select "User settings".
3. Under "Personal access tokens", click "New Token".

Then you need to create a Kubernetes Secret in your cluster to store this PAT:
```sh
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
    name: azuredevops-secret
    namespace: default # Or your desired namespace
type: Opaque
stringData:
    token: <YOUR_AZURE_DEVOPS_PAT>
EOF
```

### Create the azuredevops-system namespace

```sh
kubectl create ns azuredevops-system
```

### Set up `ConnectorConfig` for Azure DevOps Provider (classic)

After creating the secret, you can reference it from the resource `ConnectorConfig` which is used to configure the Azure DevOps provider (classic): 

```sh
kubectl apply -f - <<EOF
apiVersion: azuredevops.krateo.io/v1alpha1
kind: ConnectorConfig
metadata:
  name: connectorconfig-sample
  namespace: azuredevops-system
spec:
  apiUrl: https://dev.azure.com # DEPRECATED - use apiUrls instead
  apiVersionConfig:
    endpoints: none
    projects: 7.1-preview
  apiUrls: 
    default: https://dev.azure.com
    feeds: https://feeds.dev.azure.com
    vssps: https://vssps.dev.azure.com
  credentials:
    secretRef:
      namespace: default
      name: azuredevops-secret
      key: token
EOF
```

### Wait for Azure DevOps Provider KOG PipelinePermission controller to be ready

```sh
until kubectl get deployment azuredevops-provider-kog-pipelinepermission-controller -n krateo-system &>/dev/null; do
  echo "Waiting for PipelinePermission controller deployment to be created..."
  sleep 5
done
kubectl wait deployments azuredevops-provider-kog-pipelinepermission-controller --for condition=Available=True --namespace krateo-system --timeout=300s
```

### Set up `BasicAuth` for Azure DevOps Provider KOG

```sh
kubectl apply -f - <<EOF
apiVersion: azuredevops.kog.krateo.io/v1alpha1
kind: BasicAuth
metadata:
  name: azure-devops-basic-auth
  namespace: azuredevops-system
spec:
  username: "anything"  # Any value as official Azure DevOps OAS suggests (field not used)
  passwordRef:
    name: azuredevops-secret
    namespace: default
    key: token
EOF
```


kubectl apply -f https://raw.githubusercontent.com/vicentinileonardo/azuredevops-composition-test/refs/heads/main/compositiondefinition.yaml