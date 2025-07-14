# Azure DevOps Starter: Krateo Composition for Azure DevOps

This is a Krateo Composition that orchestrates the creation of a Azure DevOps basic setup, including a `TeamProject`, `Environment`, `GitRepository`, `Pipeline`, and `PipelinePermission`. 
It demonstrates the seamless integration of resources managed by both the Azure DevOps Provider (classic) and the Azure DevOps Provider KOG.

## Features

This Composition implements the following steps:
1.  **Creates an Azure DevOps `TeamProject`**: A new `TeamProject` is provisioned in your specified Azure DevOps organization, managed by the Azure DevOps Provider (classic).
2.  **Creates an Azure DevOps `Environment`**: An `Environment` is created within the new Team Project, managed by the Azure DevOps Provider (classic).
3.  **Creates an Azure DevOps `GitRepository`**: A `GitRepository` is created within the newly created Team Project, managed by the Azure DevOps Provider KOG.
4.  **Creates an Azure DevOps `Pipeline`**: A `Pipeline` is created in the Azure DevOps organization, which is associated with the Git Repository created in the previous step, managed by the Azure DevOps Provider KOG.
5.  **Grants Azure DevOps Pipeline Permissions**: A `PipelinePermission` resource is created to grant the necessary permissions for the Pipeline to access the Environment, managed by the Azure DevOps Provider KOG.

## Requirements

Before installing this Composition, ensure you have the following:
- **Krateo PlatformOps** installed in your Kubernetes cluster.

### Setup toolchain on krateo-system namespace

```sh
helm repo add krateo https://charts.krateo.io
helm repo update krateo
helm install azuredevops-provider krateo/azuredevops-provider --namespace krateo-system --create-namespace
helm install azuredevops-provider-kog krateo/azuredevops-provider-kog --namespace krateo-system --create-namespace
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

### Set up `ConnectorConfig` for Azure DevOps Provider (classic)

After creating the secret, you can reference it from the resource `ConnectorConfig` which is used to configure the Azure DevOps provider (classic): 

```sh
kubectl apply -f - <<EOF
apiVersion: azuredevops.krateo.io/v1alpha1
kind: ConnectorConfig
metadata:
  name: connectorconfig-sample
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

### Wait for the `RestDefinitions` of Azure DevOps Provider KOG to be ready

```sh
kubectl wait restdefinitions.swaggergen.krateo.io azuredevops-provider-kog-gitrepository --for condition=Ready=True --namespace krateo-system --timeout=300s
kubectl wait restdefinitions.swaggergen.krateo.io azuredevops-provider-kog-pipeline --for condition=Ready=True --namespace krateo-system --timeout=300s
kubectl wait restdefinitions.swaggergen.krateo.io azuredevops-provider-kog-pipelinepermission --for condition=Ready=True --namespace krateo-system --timeout=300s
```

### Create the `azuredevops-system` namespace

```sh
kubectl create ns azuredevops-system
```

### Set up `BasicAuth` for Azure DevOps Provider KOG

Note that the `BasicAuth` resource needs to be created in the `azuredevops-system` namespace, which is where the Azure DevOps Provider KOG resources will be created in the case of this Composition.

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

### Apply the `CompositionDefinition`
```sh
kubectl apply -f https://raw.githubusercontent.com/vicentinileonardo/azuredevops-starter/refs/heads/main/compositiondefinition.yaml
```

### Wait for the `CompositionDefinition` to be `Ready=True`
```sh
kubectl wait compositiondefinition azuredevops-starter --for condition=Ready=True --namespace azuredevops-system --timeout=300s
```

### Apply the `Composition`
```sh
kubectl apply -f https://raw.githubusercontent.com/vicentinileonardo/azuredevops-starter/refs/heads/main/composition.yaml
```

### Checking the status of the `Composition`

**Note**: The Composition will create multiple resources, and it may take some time for all of them to be created and become ready. With the default settings, the entire process should take approximately 10 to 15 minutes. Please note that some resources are interdependent, with relationships established through identifiers available in the status field of the resources.

#### Wait for the first 3 resources created by the Composition to be ready

**Notes**:
- you need to change the names of the resources in the commands below to match the names defined in your Composition (if you changed them).
- you need to change the namespace in the commands below to match the namespace where you applied the Composition (if you changed it).

```sh
# TeamProject
until kubectl get teamprojects.azuredevops.krateo.io project-from-composition -n azuredevops-system &>/dev/null; do
  echo "Waiting for TeamProject resource to be created..."
  sleep 5
done
echo "TeamProject resource created, waiting for TeamProject resource to be ready..."
kubectl wait teamprojects.azuredevops.krateo.io project-from-composition --for condition=Ready=True --timeout=300s

# Environment
until kubectl get environments.azuredevops.krateo.io environment-from-composition -n azuredevops-system &>/dev/null; do
  echo "Waiting for Environment resource to be created..."
  sleep 5
done
echo "Environment resource created, waiting for Environment resource to be ready..."
kubectl wait environments.azuredevops.krateo.io environment-from-composition --for condition=Ready=True --timeout=300s

# GitRepository
until kubectl get gitrepositories.azuredevops.kog.krateo.io repo-from-composition -n azuredevops-system &>/dev/null; do
  echo "Waiting for GitRepository resource to be created..."
  sleep 5
done
echo "GitRepository resource created, waiting for GitRepository resource to be ready..."
kubectl wait gitrepositories.azuredevops.kog.krateo.io repo-from-composition --for condition=Ready=True --namespace azuredevops-system --timeout=300s
```

#### Wait for the Pipeline resource to be ready

The Pipeline resource is not immediately available after applying the Composition since it needs the the ID of the GitRepository.
Therefore it will be created after the GitRepository is ready and the `status.id` field of GitRepository is populated.

```sh
until kubectl get pipelines.azuredevops.kog.krateo.io pipeline-from-composition -n azuredevops-system &>/dev/null; do
  echo "Waiting for Pipeline resource to be created..."
  sleep 5
done
echo "Pipeline resource created, waiting for Pipeline resource to be ready..."
kubectl wait pipelines.azuredevops.kog.krateo.io pipeline-from-composition --for condition=Ready=True --namespace azuredevops-system --timeout=300s
```

#### Wait for the PipelinePermission resource to be ready

The PipelinePermission resource is created after the Pipeline resource is ready, as it needs the IDs of both the Pipeline and Environment.

```sh
until kubectl get pipelinepermissions.azuredevops.kog.krateo.io pipelinepermission-from-composition -n azuredevops-system &>/dev/null; do
  echo "Waiting for PipelinePermission resource to be created..."
  sleep 5
done
echo "PipelinePermission resource created, waiting for PipelinePermission resource to be ready..."
kubectl wait pipelinepermissions.azuredevops.kog.krateo.io pipelinepermission-from-composition --for condition=Ready=True --namespace azuredevops-system --timeout=300s
```

### Cleanup

```sh
kubectl delete -f https://raw.githubusercontent.com/vicentinileonardo/azuredevops-starter/refs/heads/main/composition.yaml

kubectl delete -f https://raw.githubusercontent.com/vicentinileonardo/azuredevops-starter/refs/heads/main/compositiondefinition.yaml
```
