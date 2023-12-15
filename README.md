# Step by step guid to use FluxCD AKS Extension

## Introduction

This guide will help you to use FluxCD AKS Extension to deploy your application to AKS cluster.

The deployed application is [AKS Store Demo](https://github.com/Azure-Samples/aks-store-demo)

To start from the beginning, clone or fork this repo, then delete all files except README.md and manifests folder. Then follow the steps below.

## Prerequisites

- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [GitHub Account](https://github.com/)
- [Flux CLI](https://fluxcd.io/docs/installation/#install-the-flux-cli)
- [Kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/)

## Create AKS Cluster

Create AKS cluster using Azure CLI

```bash
$rgName="flux-rg"
$aksClusterName="flux-aks"
$location="swedencentral"

az group create --name $rgName --location $location
az aks create --resource-group $rgName --name $aksClusterName  --enable-addons monitoring --enable-oidc-issuer --enable-workload-identity
```

Get AKS cluster credentials

```bash
az aks get-credentials --resource-group $rgName --name $aksClusterName
```

Before you can install the FluxCD AKS extension using Azure CLI, you need to make sure that you have the proper extensions installed.

```bash
az extension add --name aks-preview
az extension add --name k8s-extension
az extension add --name k8s-configuration
```

## Install FluxCD AKS Extension

With the Azure CLI extensions for Kubernetes installed, you can now install the FluxCD AKS extension using the following command.

```bash
az k8s-extension create --cluster-name $aksClusterName --resource-group $rgName --cluster-type managedClusters --extension-type microsoft.flux --name aks-store-demo
```

With the extension installed, you can run the following Flux CLI command to get the status of the installation.

```bash
flux check --pre
```

Flux installs many new Custom Resource Definitions (CRDs) in the cluster. These CRDs are how you interact with FluxCD. You can run the following command to see all the new CRDs

```bash
kubectl get crds | grep flux
```

## Prepare Kubernetes manifests for Kustomize

In the manifests folder, you will find the manifest files. Now you need to create a Kustomization file to tell FluxCD how to deploy the manifests.
This will be done using the kustomize CLI.

```bash
cd manifests
kustomize create --autodetect
```

## Create a `dev`´overlay

Next, we need to create an overlay for our dev environment.

```bash
cd ..
mkdir -p overlays/dev
cd overlays/dev
```

We want our dev deployment to deploy to a new namespace so we'll create a new manifest to create one. Run the following command to create a new manifest file.

```bash
kubectl create namespace store-dev --dry-run=client -o yaml > namespace.yaml
```

Generate the kustomization.yaml file for our dev overlay using the following command.

```bash
kustomize create --resources namespace.yaml,./../../manifest --namespace store-dev
```

In this kustomization.yaml file, we see that it will include our new namespace.yaml file and all manifests in the manifest directory. We also have a namespace: store-dev entry in the kustomization and this is what instructs Kustomize to patch and add the store-dev namespace to the base manifests.

## Kustomize in action

Let's test the dev overlay by running the following command from the dev directory

```bash
kustomize build
```

Notice how all the manifests are patched with the store-dev namespace and output to the console. This is how Kustomize works. It patches the base manifests with environment specific configurations and outputs the patched manifests to the console.

To deploy these patched manifests, you would run a command like this:

```bash
kustomize build | kubectl apply -f -
```

Kustomize is also built into kubectl so you can run the following command to apply the manifests to your cluster.

```bash
kubectl apply -k .
```

Hopefully you didn't run the commands above. If you did, no worries, you can delete this deployment with either of these commands.

```bash
# using kustomize
kustomize build | kubectl delete -f -

# or using kubectl
kubectl delete -k .
```

## Deploying Applications with FluxCD

FluxCD will monitor the repo for changes and reconcile the cluster with the desired state defined in the repo.

Run the following command to get the GitHub HTTP URL for your repo

```bash
$ghRepoUrl=$(gh repo view --json url | jq -r '.url')
```

Using Azure CLI again, let's configure the FluxCD AKS extension to connect to our GitHub repo.

```bash
az k8s-configuration flux create --cluster-name $aksClusterName --resource-group $rgName --cluster-type managedClusters --name aks-store-demo --url $ghRepoUrl --branch main --kustomization name=dev path=./overlays/dev --namespace flux-system
```

The above command is telling FluxCD AKS extension to connect to our GitHub repo and monitor the `main` branch for changes. When changes are detected, FluxCD will apply the changes to the cluster using the Kustomization defined in the dev overlay. Lastly, we tell Flux to create new Flux GitRepository and Kustomization resources in the flux-system namespace. You can change this to whatever namespace you want.

If you run the following Flux CLI commands you should see some resources created.

```bash
flux get sources git
flux get kustomizations
```

If all went well, you should see your pods coming online. Let's watch for them:

```bash
watch kubectl get pods -n store-dev -w
```

Let's test the application by grabbing the public IP address of the store service:

```bash
kubectl get svc/store-front -n store-dev
```

Open a browser and navigate to the IP address. You should see the store front page.

## Making and managing changes

With the FluxCD AKS extension installed and connected to our GitHub repo, we can now make changes by simply editing the kubernetes manifests and committing/pushing the changes back to the remote repo. At this point, it's all about Git workflows and processes.

Let's make a small change to the dev overlay kustomization.yaml file. Let's say we want to change the name of the namespace from store-dev to just dev.

Open the overlays/dev/kustomization.yaml file and change the namespace value from store-dev to dev.

Compare the changes:

```bash
git diff overlays/dev/kustomization.yaml
```

Add the change, commit, and push to GitHub.

Using the Flux CLI, you can force FluxCD to reconcile the cluster with the desired state defined in the repo.

```bash
flux reconcile kustomization aks-store-demo-dev --with-source
```

fter a minute or two you should see the pods coming online in the new dev namespace. This is FluxCD reconciling the cluster with the desired state defined in the repo.

You can check on the pods using the following command.

```bash
kubectl get pods -n dev
```

## Monitoring and Troubleshooting

Some tips when it comes to monitoring and troubleshooting Flux resources is to use the Flux CLI. You can use some of these basic commands to get information about Flux and its resources:

```bash
# check the status of the flux installation
flux check

# get info about the GitRepository resource
flux get source git aks-store-demo-dev -n flux-system

# get info about the Kustomization resource
flux get kustomization aks-store-demo-dev -n flux-system

# view event logs from the flux controllers
flux events

# view logs from the flux controllers
flux log

# view stats of the flux controllers
flux stats
```

This showed the setup of FluxCD AKS Extension and how to use it to deploy applications to AKS cluster. Now we will move a bit deeper and how to use FluxCD to automate images updates.
[Image updates](./IMAGE_UPDATES_README.md)