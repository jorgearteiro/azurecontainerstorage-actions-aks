# Using Azure Container Store and Azure Files Premium with GitHub Actions and Kubernetes Tutorial

On this tutorial, you can learn how to use Azure Container Storage to Ephemeral volumes and Azure File share Premium to Persistent volumes. These Azure services will be used as storage in GitHub Self Hosted runners running on AKS - Azure Kubernetes services. We are using GitHub ARC - Actions Runner Controller and ARC Runner Scale sets for Kubernetes to manage and scale your self-hosted github runners.

Azure Files share allows a volume to be mounted in multiple AKS pods and it's great for caching. and Azure Container Storage have a great, replicated and fast NVMe Ephemeral disks/volume creation that can be used as temporary disks for Github Actions jobs.

![Solution Architecture](./docs/images/Tutorial_Solution_Architecture.png)

Learn more here about [GitHub ARC](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/about-actions-runner-controller), [Azure File share](https://learn.microsoft.com/en-us/azure/aks/azure-files-csi) and [Azure Container Storage](https://learn.microsoft.com/en-us/azure/storage/container-storage/]) running on Kubernetes.

## Pre-requisites

We are using 3 CLI tools: Azure CLI, Kubectl and Helm. If you are running in CloudShell, these tools are already available there for you.

* Install [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli#install-or-update)
* Install [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/#install-kubectl-binary-with-curl-on-windows)
* Install [Helm](https://helm.sh/docs/intro/install/)

## Defining parameters

Make sure to replace the following mandatory placeholders :

* `AKS_AND_STORAGE_ACCOUNT_RG` with the name of the resource group used by storage account and AKS cluster
* `AKS_CLUSTER_NAME` with the name of the AKS - Azure Kubernetes Services cluster to be created
* `STORAGE_ACCOUNT_NAME` with the name of the storage account to be created
* `AKS_STORAGE_ACCOUNT_LOCATION` with the name of the region to create the resources in. we are deploying on the same region as the AKS cluster nodes to facilitate performance and cost.
* `GITHUB_CONFIG_URL` the URL to GitHub organisation or repository

and these are optional, please keep these default values if possible :

* `NAMESPACE_ARC_CONTROLLER` the name of Kubernetes namespace to run Arc runners scaleset controller
* `ARC_CONTROLLER_NAME` the name of Arc runners scaleset controller
* `NAMESPACE_ARC_RUNNERS` the name of Kubernetes namespace to run Arc self-hosted runners
* `ARC_RUNNER_SCALESET_NAME` the name of Arc runners scaleset
* `ARC_RUNNER_GITHUB_SECRET_NAME` the name of GITHUB secret

```bash
# please fill these env variables with your details
AKS_AND_STORAGE_ACCOUNT_RG="aks-storage-actions"
AKS_CLUSTER_NAME="aks-storage-actions"
STORAGE_ACCOUNT_NAME="fileshareactions"
AKS_STORAGE_ACCOUNT_LOCATION="westus3"
GITHUB_CONFIG_URL="https://github.com/jorgearteiro/azurecontainerstorage-actions-aks"

# optional, changes maybe require aditional chnages on ./install/*.yaml files
NAMESPACE_ARC_CONTROLLER="arc-systems-storage"
ARC_CONTROLLER_NAME="arc-controller-storage"
NAMESPACE_ARC_RUNNERS="arc-runners-storage"
ARC_RUNNER_SCALESET_NAME="arc-runner-set-storage"
ARC_RUNNER_GITHUB_SECRET_NAME="arc-runner-github-secret"
```

## Create AKS - Azure Kubernetes Services Cluster

Please follow the [Quickstart: Deploy an Azure Kubernetes Services (AKS) cluster using Azure CLI](https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-cli) to create the required Azure Kubernetes Services.

Run the following command to create your AKS Cluster:

```bash
# Create Resource Group used by AKS and Storage account
az group create --name "${AKS_AND_STORAGE_ACCOUNT_RG}" --location "${AKS_STORAGE_ACCOUNT_LOCATION}"

# Create AKS Cluster
az aks create -g "${AKS_AND_STORAGE_ACCOUNT_RG}" -n "${AKS_CLUSTER_NAME}" \
       --os-sku AzureLinux \
       --node-count 3 \
       --enable-azure-container-storage ephemeralDisk \
       --storage-pool-option NVMe \
       --node-vm-size Standard_L8s_v3 \
       --max-pods=250 \
       --network-plugin azure \
       --network-plugin-mode overlay \
       --generate-ssh-keys
```

To manage a Kubernetes cluster, use the Kubernetes command-line client, [kubectl][kubectl]. `kubectl` is already installed if you use Azure Cloud Shell. To install `kubectl` locally, use the `az aks install-cli` command.

1. Configure `kubectl` to connect to your Kubernetes cluster using the `az aks get-credentials` command. This command downloads credentials and configures the Kubernetes CLI to use them.

    ```bash
    az aks get-credentials -g "${AKS_AND_STORAGE_ACCOUNT_RG}" -n "${AKS_CLUSTER_NAME}"
    ```

1. Verify the connection to your cluster using the `kubectl get` command. This command returns a list of the cluster nodes.

    ```bash
    kubectl get nodes
    ```

## Create an Azure Container Storage Ephemeral Storage pool with Replicas

[Azure Container Storage](https://learn.microsoft.com/en-us/azure/storage/container-storage/]) creates a default storage pool on the AKS cluster creation command using `--enable-azure-container-storage ephemeralDisk` flag. We are going to remove this pool and create another one with 3 replicas using Kubernets CRDs.

```bash
# delete the default ephemeral storage pool create on the cluster creation
kubectl delete sp -n acstor ephemeraldisk-nvme

# Create new ephemeral storage pool with 3 replicas using Kubernetes CRDs
kubectl apply -f - <<EOF
apiVersion: containerstorage.azure.com/v1
kind: StoragePool
metadata:
  name: ephemeraldisk-nvme
  namespace: acstor
spec:
  poolType:
    ephemeralDisk:
      diskType: nvme
      replicas: 3
EOF
```

This command will generate a Storage Class called `acstor-ephemeraldisk-nvme` that can now be used as your Ephemeral to create your Self Hosted runners Ephemeral temporary disks when running Github Actions jobs.

After a minute you can check the storage pool creation running this command.

```bash
kubectl get sp -n acstor -o json
```

## Create an Azure file share

Before you can use an Azure Files file share as a Kubernetes volume, you must create an Azure Storage account and the file share. We are using Azure file share Premium SMB with support for metadata caching. The minimal is 100 Gb for each share you create.

1. Create a storage account using the `az storage account create` command with the `--sku` parameter. The following command creates a storage account using the `Premium_LRS` SKU.

    ```bash
    az storage account create -n "${STORAGE_ACCOUNT_NAME}" -g "${AKS_AND_STORAGE_ACCOUNT_RG}" -l "${AKS_STORAGE_ACCOUNT_LOCATION}" --sku Premium_LRS --kind FileStorage
    ```

2. Export the connection string as an environment variable using the following command, which you use to create the file share.

    ```bash
    export AZURE_STORAGE_CONNECTION_STRING=$(az storage account show-connection-string -n ${STORAGE_ACCOUNT_NAME} -g ${AKS_AND_STORAGE_ACCOUNT_RG} -o tsv)
    ```

3. Create the 100Gb premium file share using the `az storage share create` command. We are using `metadatacaching` as share name. If you change this name, you also have to change `arc-runners-set-storage-pv-pvc.yaml` file to reflect this change.

    ```bash
    az storage share create -n metadatacaching --quota 100 --connection-string $AZURE_STORAGE_CONNECTION_STRING
    ```

## Installing ARC Runners Scaleset Controler

```bash
helm install "${ARC_CONTROLLER_NAME}" \
    --namespace "${NAMESPACE_ARC_CONTROLLER}" \
    --create-namespace \
    --version "0.9.3" \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

Please remove the `--version "0.9.3"` parameter to install the latest version. The Arc runner-set need to have the same version of the Arc controler.

## Creating Kubernetes Secrets

### Azure File Share Storage Key Secret

Azure Files requires a secret to be created on AKS with the Storage key used to connect the Azure File share from the AKS pod container.

```bash
STORAGE_KEY=$(az storage account keys list --resource-group ${AKS_AND_STORAGE_ACCOUNT_RG} --account-name ${STORAGE_ACCOUNT_NAME} --query "[0].value" -o tsv)

kubectl create namespace "${NAMESPACE_ARC_RUNNERS}"

kubectl create secret generic azure-storage-secret \
   --namespace "${NAMESPACE_ARC_RUNNERS}" \
   --from-literal=azurestorageaccountname=${STORAGE_ACCOUNT_NAME} \
   --from-literal=azurestorageaccountkey=${STORAGE_KEY} 
```

### GitHub App Secret

Create a GitHub App to allow the self-hosted runner to access your GitHub organisation or repository. Please follow the instructions [here](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app)

These are the parameters provide by GitHub App creation process:

* `GITHUB_APP_ID` Github App ID created
* `GITHUB_APP_INSTALLATION_ID` GitHub App Installation ID created
* `github_app_private_key` replace the whole '-----BEGIN RSA PRIVATE KEY----- section with your Private key

```bash
GITHUB_APP_ID=856120
GITHUB_APP_INSTALLATION_ID=48447618

kubectl create secret generic ${ARC_RUNNER_GITHUB_SECRET_NAME} \
   --namespace=${NAMESPACE_ARC_RUNNERS} \
   --from-literal=github_app_id=${GITHUB_APP_ID} \
   --from-literal=github_app_installation_id=${GITHUB_APP_INSTALLATION_ID} \
   --from-literal=github_app_private_key='-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA86Cfc3qBK0EiLtFMGVaTGydZc9NuBSir0I1G6iqRXV5bp40N
1ya3v/PMWWnriq8uX2ThZodBBTbD9A8CA/GTuYdUVhWGluACMjJHiQXBB77okwWT
cz7oUffPYGbwW9koA8h7yU2HR3yIvb82ZdNPrOAg/GPJLILZ4WvoWXq2DrmPb4+K
pN3NxBN6DeuUE2NsdfCxXybRsbQr3sEuvpaffHkUIkjBwnMzFJjQV8H4QbNt+ut4
eV1l368TxaPZMbx0YTuoBxCMhFj2NRLUNObDixK/xZFSpgvxT10wR17ak8WZNnLx
cz7oUffPYGbwW9koA8h7yU2HR3yIvb82ZdNPrOAg/GPJLILZ4WvoWXq2DrmPb4+K
pN3NxBN6DeuUE2NsdfCxXybRsbQr3sEuvpaffHkUIkjBwnMzFJjQV8H4QbNt+ut4
eV1l368TxaPZMbx0YTuoBxCMhFj2NRLUNObDixK/xZFSpgvxT10wR17ak8WZNnLx
cz7oUffPYGbwW9koA8h7yU2HR3yIvb82ZdNPrOAg/GPJLILZ4WvoWXq2DrmPb4+K
pN3NxBN6DeuUE2NsdfCxXybRsbQr3sEuvpaffHkUIkjBwnMzFJjQV8H4QbNt+ut4
eV1l368TxaPZMbx0YTuoBxCMhFj2NRLUNObDixK/xZFSpgvxT10wR17ak8WZNnLx
cz7oUffPYGbwW9koA8h7yU2HR3yIvb82ZdNPrOAg/GPJLILZ4WvoWXq2DrmPb4+K
pN3NxBN6DeuUE2NsdfCxXybRsbQr3sEuvpaffHkUIkjBwnMzFJjQV8H4QbNt+ut4
eV1l368TxaPZMbx0YTuoBxCMhFj2NRLUNObDixK/xZFSpgvxT10wR17ak8WZNnLx
cz7oUffPYGbwW9koA8h7yU2HR3yIvb82ZdNPrOAg/GPJLILZ4WvoWXq2DrmPb4+K
pN3NxBN6DeuUE2NsdfCxXybRsbQr3sEuvpaffHkUIkjBwnMzFJjQV8H4QbNt+ut4
eV1l368TxaPZMbx0YTuoBxCMhFj2NRLUNObDixK/xZFSpgvxT10wR17ak8WZNnLx
cz7oUffPYGbwW9koA8h7yU2HR3yIvb82ZdNPrOAg/GPJLILZ4WvoWXq2DrmPb4+K
pN3NxBN6DeuUE2NsdfCxXybRsbQr3sEuvpaffHkUIkjBwnMzFJjQV8H4QbNt+ut4
eV1l368TxaPZMbx0YTuoBxCMhFj2NRLUNObDixK/xZFSpgvxT10wR17ak8WZNnLx
cz7oUffPYGbwW9koA8h7yU2HR3yIvb82ZdNPrOAg/GPJLILZ4WvoWXq2DrmPb4+K
pN3NxBN6DeuUE2NsdfCxXybRsbQr3sEuvpaffHkUIkjBwnMzFJjQV8H4QbNt+ut4
eV1l368TxaPZMbx0YTuoBxCMhFj2NRLUNObDixK/xZFSpgvxT10wR17ak8WZNnLx
pN3NxBN6DeuUE2NsdfCxXybRsbQr3sEuvpaffHkUIkjBwnMzFJjQV8H4QbNt+ut4
s9uqYckJaMLIY6J2lRmodK9ybknmIJt/ji5R1ugBqF9hlW429tSnJg==
-----END RSA PRIVATE KEY-----
'
```

## Azure File share configurations

Azure Files fileshare can be mounted in multiple pods at same time. We can use this capability called AcccessMode: ReadWriteMany to mount the same fileshare in all pods created by the Arc kubernetes replicateset.

As a persistent SMB File share to cache Nuget packages used by our .NET example Application. The [`arc-runners-set-storage-pv-pvc.yaml`](./install/arc-runners-storage-set-pv.yaml) file will create the required PV and PVC for this File Share. We recomend Azure File Premium for this first option.
Please customize `volumeAttributes` and any `namespaces` parameter on both PV - Persistent Volume and PVC - Persistent volume claim manifests as showed here:

```yaml
volumeAttributes:
  resourceGroup: aks-storage-actions  # optional, only set this when storage account is not in the same RG group as node
  shareName: metadatacaching
nodeStageSecretRef:
  name: azure-storage-secret
  namespace: arc-runners-storage      
```

```bash
# Create PV and PVC on your cluster
kubectl apply -f ./install/arc-runners-set-storage-pv-pvc.yaml --namespace "${NAMESPACE_ARC_RUNNERS}" --wait
```

## Installing ARC Runner Scale Set

Install ARC Runner Scale Set using the official GitHub Helm chart and manually mount your Azure Files share on Kubernetes

This is a code snippet from the [`arc-runners-set-storage-values.yaml`](./install/arc-runners-set-storage-values.yaml) file on the Install folder that can be customized before installing the Runner set Helm Chart. 

We are using a customized version the `Kubernetes` containerMode, to include Azure File share volume mountings to Nuget packages and to ephemeral _work folder volume.

The only not mandantory changes are:

* `storageClassName` choose between "github-azurefile-premium" and "github-azurefile"
* `storage` choose the size of the storage. 100Gb minimum to premium.

The other helm parameters will be set on the helm install command using --set option.

```yaml
containerMode:
  type: "kubernetes"  ## type can be set to dind or kubernetes
  ## the following is required when containerMode.type=kubernetes
  kubernetesModeWorkVolumeClaim:
    accessModes: ["ReadWriteOnce"]
    storageClassName: "acstor-ephemeraldisk-nvme" # this is the Azure Container Storage Ephemeral pool class
    resources:
      requests:
        storage: 5Gi # Customiza as required

template:
  spec:
  securityContext:
    fsGroup: 123 # Group used by GitHub default agent image
  containers:
  - name: runner
    image: ghcr.io/actions/actions-runner:latest
    command: ["/home/runner/run.sh"]
    env:
      - name: ACTIONS_RUNNER_REQUIRE_JOB_CONTAINER
        value: "false"
      - name: ACTIONS_RUNNER_CONTAINER_HOOK_TEMPLATE
        value: "/home/runner/container-config/container-podspec.yaml"
    volumeMounts:
      - name: "container-podspec-volume"
        mountPath: "/home/runner/container-config"
      - name: azurefile
        mountPath: /home/runner/.nuget/             
  volumes:
    - name: "container-podspec-volume"
      configMap:
        name: hook-extension
    - name: azurefile
      persistentVolumeClaim:
        claimName: azurefile
```

For compatibility with GitHub Workflow container feature that allows you to run containers inside your pipeline, we are mounting a `container-podspec-volume` with the pod spec for the workflow pod created by ARC when running workflows with the container feature. This pod spec is mounted from a config map created on `arc-runners-set-container-pod-spec.yaml` file on the install folder. No changes are required.

```bash
kubectl apply -f ./install/arc-runners-set-container-pod-spec.yaml
```

### ARC Runner Scaleset Helm Chart Parameters

kubectl apply -f ./install/arc-runners-set-container-pod-spec.yaml

The Arc Runner Scaleset Helm Chart provides a few parameters, these are the most important ones to install a Scaleset with Azure File share volume mount on AKS - Azure Kubernetes Services.

These are the parameters:

* `githubConfigUrl` your Github Organisation or repository. We are using repository on our example.
* `githubConfigSecret` the GitHub App Secret to access Github from the self-hosted runner
* `minRunners` Minimal number of runners on the scale set, waiting for new jobs from GitHub
* `maxRunners` Maximal number of runners, running jobs or waiting for new jobs from GitHub
* `runnerGroup` Github runner Group supported used by the Arc runnerset.

Arc runner set helm chart is available to download here `oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set`

To instal the Helm Chart on AKS, please run "helm install" command on your AKS Cluster

```bash
helm install "${ARC_RUNNER_SCALESET_NAME}" \
    --namespace "${NAMESPACE_ARC_RUNNERS}" \
    --create-namespace \
    --values ./install/arc-runners-set-storage-values.yaml \
    --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
    --set githubConfigSecret="${ARC_RUNNER_GITHUB_SECRET_NAME}" \
    --set minRunners=1 \
    --set maxRunners=3 \
    --set runnerGroup=default \
    --version "0.9.3" \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

Please remove the `--version "0.9.3"` parameter to install the latest version. The Arc runner-set need to have the same version of the Arc controler.

### Upgrading a Runner scale set installation

If you want to upgrade any configuration on the Arc Runner Scaleset, re-run the last helm install command onyl replacing the firt line to "helm upgrade --install".

```bash
helm upgrade --install "${ARC_RUNNER_SCALESET_NAME}" \
    --namespace "${NAMESPACE_ARC_RUNNERS}" \
    --create-namespace \
    --values ./install/arc-runners-set-storage-values.yaml \
    --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
    --set githubConfigSecret="${ARC_RUNNER_GITHUB_SECRET_NAME}" \
    --set minRunners=1 \
    --set maxRunners=3 \
    --set runnerGroup=default \
    --version "0.9.3" \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

Please remove the `--version "0.9.3"` parameter to install the latest version. The Arc runner-set need to have the same version of the Arc controler.

## Running your Workflows - GitHub Actions

I have created 3 workflows on this repository, under the default GitHub workflow folder `.github/workflows` for you to test the self-hosted ARC runners created on AKS.

* `.NET Build using containers` install .NET SDK and restore/build/publish application on the runner itself. File name is `dotnet-using-container.yml`
* `.NET Build without containers` use workflow container feature to run a .NET SDK container and build inside the application inside the container. NUGET Caching is mounted by default on this container. File name is `dotnet-wihout-container.yml`
* `Container and Service Test` testing workflows also using containers feature to create a ubuntu container and a redis service. Both containers run on the same AKS Pod. NUGET Caching is also mounted by default on this container. File name is `container-service-test.yml`

All 3 workflows have an input parameter for the Arc runner name to be used on the `runs-on:` field of your workflow. This is the `ARC_RUNNER_SCALESET_NAME="arc-runner-set"` variable defined before, called `arc-runner-set-storage`. To facilitate testing, we are using `workflow_dispatch:` option on the 3 workflows to only run those workflows when it is requested manually. On GitHub Actions tab of your repository, select one of the workflows and click on `Run worflow` button.

Once the workflow is running, it will request a runner to ARC running on AKS cluster. Once this runner, a pod on Kubernetes, is allocated for the job, the workflow will run in there to completion. As we are using the Ephemeral runner approach, the pod running your workflow will be destroyed at the end and a new one will be created for your next workflow run.

## Removing resources

If you want to remove all resources created on your AKS - Azure Kubernetes Cluster, run these commands.

```bash
# Deleting ARC Runner Scalesets
helm delete "${ARC_RUNNER_SCALESET_NAME}" -n "${NAMESPACE_ARC_RUNNERS}" --wait

# Deleting ARC Runners Scaleset Controler
helm delete "${ARC_CONTROLLER_NAME}" -n "${NAMESPACE_ARC_CONTROLLER}" --wait

# Delete Azure File share configurations
kubectl delete -f ./install/arc-runners-set-storage-pv-pvc.yaml --wait

# Delete secrets
kubectl delete secret azure-storage-secret -n arc-runners --wait
kubectl delete secret ${ARC_RUNNER_GITHUB_SECRET_NAME} -n arc-runners --wait

# Delete container runner configmap pod spec
kubectl delete -f .install/arc-runners-set-container-pod-spec.yaml --wait

# Deleting Namespaces
kubectl delete namespace ${NAMESPACE_ARC_RUNNERS}
kubectl delete namespace ${NAMESPACE_ARC_CONTROLLER}
```

If you don't plan on going through this guideline, clean up unnecessary resources to avoid Azure charges. Remove the resource group, AKS container service, Azure File share and all related resources.
