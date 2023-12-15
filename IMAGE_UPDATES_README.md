# Image Updates

The objective of this lab is to demonstrate how to use FluxCD to automate image updates in a Kubernetes cluster. The lab will use the [AKS Store Demo](https://github.com/Azure-Samples/aks-store-demo) application as an example.

The intended workflow:
1. Modify application code, then commit and push the change to the repo.
2. Create a new release in GitHub which kicks off a release workflow to build and push an updated container image to a GitHub Container Registry.
3. FluxCD detects the new image and updates the image tag in the cluster.
4. FluxCD rolls out the new image to the cluster.

## Create AKS Cluster

Create AKS cluster using Azure CLI

```bash
$rgName="flux-rg"
$aksClusterName="imageupdate-aks"
$location="swedencentral"

az group create --name $rgName --location $location
az aks create --resource-group $rgName --name $aksClusterName  --enable-addons monitoring --enable-oidc-issuer --enable-workload-identity
```

Get AKS cluster credentials

```bash
az aks get-credentials --resource-group $rgName --name $aksClusterName
```

## Bootstrapping FluxCD

In the first section I used the AKS extension to install FluxCD. In this section I will use the Flux CLI to install FluxCD. This will enable us to have a bit more control over the installation and gives the ability yo save Flux resources in the repo.

Ensure the cluster is ready and set the following environment variables:

```bash
flux check --pre
$ghUser=$(gh api user --jq .login)
 
$ghToken=$(gh auth token) 
```

Now we're ready to bootstrap FluxCD. Run the following command to bootstrap FluxCD. This command will install FluxCD with additional components to enable image automation. It will also generate the Flux resources and commit them to our Git repo in the clusters/dev directory.

```bash
echo $ghToken | flux bootstrap github create --owner=$ghUser --repository=gitops --personal --path=./clusters/dev --branch=main --reconcile --network-policy --components-extra=image-reflector-controller,image-automation-controller
```

In order for the image-automation-controller to write commits to our repo, we need to create a Kubernetes secret to store our GitHub credentials. Run the following command to create the secret.

```bash
flux create secret git aks-store-demo --url=$ghRepoUrl --username=$ghUser --password=$ghToken
```

This command will create Kubernetes secrets in your cluster but you could also use Azure Key Vault with the Secret Store CSI driver to store your GitHub credentials.

Run the following command to create the configuration and export it to a YAML file which we'll commit to our repo.

```bash
flux create source git aks-store-demo --url=$ghRepoUrl --branch=main --interval=1m --secret-ref=aks-store-demo --export > ./clusters/dev/aks-store-demo-source.yaml
```