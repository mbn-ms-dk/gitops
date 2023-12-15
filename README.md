# Step by step guid to use FluxCD AKS Extension

## Introduction

This guide will help you to use FluxCD AKS Extension to deploy your application to AKS cluster.

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
flux check
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
