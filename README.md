# Guide to to automate image updates with FluxCD on AKS

The intended workflow:

1. Create the application
2. Test locally
3. Commit and push to GitHub
4. push image to Azure Container Registry
5. Kustomize the manifests

## Introduction

This guide will help you to use FluxCD to deploy your application to AKS cluster.

## Prerequisites

- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- [GitHub Account](https://github.com/)
- [Flux CLI](https://fluxcd.io/docs/installation/#install-the-flux-cli)
- [Kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/)


## Create the application

Create a `src` directory and use dotnet new to create blazor server app.

```bash
mkdir src
cd src
dotnet new blazorserver -o MyApp
```

Navigate to the app directory and run the app

```bash
cd MyApp
dotnet run
```

You should see the app running on http://localhost:5073

![MyApp](./images/initrun.png)

Navigate to root and use dotnet new to create a new solution file.

```bash
cd ..
dotnet new sln 
```

Add MyApp to the solution

```bash
dotnet sln add src/MyApp/MyApp.csproj
```

Create a dockerfile for the app

## Create an Azure Container Registry

```bash	
$rgName="ffi-rg"
$aksClusterName="ffi-aks"
$location="swedencentral"
$acrName="ffiacr"
```	

Create a resource group for the Azure Container Registry and create the Azure Container Registry.

```bash
az group create --name $rgName --location $location
az acr create --resource-group $rgName --name $acrName --sku Basic
```

Build and push the image to the Azure Container Registry.

```bash
az acr build --registry $acrName -t myapp:v1 -f ./src/MyApp/dockerfile .
```

## Create AKS Cluster

Create AKS cluster using Azure CLI and attach the Azure Container Registry to the AKS cluster.

```bash
az group create --name $rgName --location $location
az aks create --resource-group $rgName --name $aksClusterName  --enable-addons monitoring --attach-acr $acrName
```

Get AKS cluster credentials

```bash
az aks get-credentials --resource-group $rgName --name $aksClusterName
```

Get your login server address using the az acr list command and query for your login server.

```bash
az acr list --resource-group $rgName --query "[].{acrLoginServer:loginServer}" --output table
```

Set the image property for the containers in the manifests/myapp.yaml file.

Deploy the app to AKS cluster

```bash
kubectl apply -f manifests/myapp.yaml
```

Test the app

```bash
kubectl get svc myapp
```
The output should look like this:

![MyApp](./images/testonaks.png)

## Prepare Kubernetes manifests for Kustomize

Kustomize is not a templating engine like Helm, it is more like a patching engine. It allows you to create a base set of Kubernetes manifests and then patch them with environment specific configurations. These environmental configurations are known as "overlays". So you can have a `dev` overlay, a `prod` overlay, a `staging` overlay, etc. and use them to patch the base manifests with environment specific configurations.

Use the Kustomize CLI, run the following command to create the base kustomization.yaml file:

```bash
cd manifests
kustomize create --autodetect
```

## Create a `dev` overlay

Next, we need to create an overlay for our dev environment.

We need to navigate back to the root of the repo directory and create a dev overlay directory.

```bash
cd ../
mkdir -p overlays/dev
cd overlays/dev
```

We want our dev deployment to deploy to a new namespace so we'll create a new manifest to create one. Run the following command to create a new manifest file.

```bash
kubectl create namespace myapp-dev --dry-run=client -o yaml > namespace.yaml
```
Generate the kustomization.yaml file for our dev overlay using the following command.

```bash
kustomize create --resources namespace.yaml,./../../manifests --namespace myapp-dev
```

Let's test the `dev` overlay by running the following command from the `dev` directory:

```bash
kustomize build
```

Notice how all the manifests are patched with the `myapp-dev` namespace and output to the console. This is how Kustomize works. It patches the base manifests with environment specific configurations and outputs the patched manifests to the console.

To deploy these patched manifests, you would run a command like this.

```bash
kustomize build | kubectl apply -f -
```

Hopefully you didn't run the command above. If you did, then you can delete this deployment with either of these commands.

```bash
# using kustomize
kustomize build | kubectl delete -f -

# or using kubectl
kubectl delete -k .
```

## Bootstrapping FluxCD

Use the Flux CLI to bootstrap FluxCD onto the AKS cluster. This process will enable us to have a bit more control over the installation and gives us the ability to save the Flux resources to our GitHub repo.

First, we need the [Flux CLI](https://fluxcd.io/flux/installation/). Once you have the CLI installed, run the following command to ensure your cluster is ready for bootstrapping.

```bash
flux check --pre
```

We need to set some environment variables to for the Flux bootstrapping process. Run the following commands to set the environment variables (I switch to WSL here).

```bash
# set the repo url
export GITHUB_REPO_URL=$(gh repo view --json url | jq .url -r)

# set your GitHub username
export GITHUB_USER=$(gh api user --jq .login)

# set your GitHub personal access token
export GITHUB_TOKEN=$(gh auth token)
```

Now we're ready to bootstrap FluxCD. Run the following command to bootstrap FluxCD. This command will install FluxCD with additional components to enable image automation. It will also generate the Flux resources and commit them to our Git repo in the `clusters/dev` directory.

```bash
flux bootstrap github create \
  --owner=$GITHUB_USER \
  --repository=gitops \
  --personal \
  --path=./clusters/dev \
  --branch=main \
  --reconcile \
  --network-policy \
  --components-extra=image-reflector-controller,image-automation-controller
```

In order for the `image-automation-controller` to write commits to our repo, we need to create a Kubernetes secret to store our GitHub credentials. Run the following command to create the secret:

```bash
flux create secret git myapp \
  --url=$GITHUB_REPO_URL \
  --username=$GITHUB_USER \
  --password=$GITHUB_TOKEN
```

We don't need the GitHub PAT token anymore, so run the following command to discard it.

```bash
unset GITHUB_TOKEN
```

Next we need to create a `GitRepository` resource.
Run the following command to create the configuration and export it to a YAML file which we'll commit to our repo:

```bash
flux create source git myapp \
  --url=$GITHUB_REPO_URL \
  --branch=main \
  --interval=1m \
  --secret-ref=myapp \
  --export > ./clusters/dev/myapp-source.yaml
```

We also need to specify the Kustomization resource to tell FluxCD where to find the app deployment manifests in our repo. Run the following command to create the configuration and export it to a YAML file which we'll also commit to our repo.

```bash
flux create kustomization myapp \
  --source=gitops \
  --path="./overlays/dev" \
  --prune=true \
  --wait=true \
  --interval=1m \
  --retry-interval=2m \
  --health-check-timeout=3m \
  --export > ./clusters/dev/myapp-kustomization.yaml
```
