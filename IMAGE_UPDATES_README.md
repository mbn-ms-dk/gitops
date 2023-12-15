# Image Updates

The objective of this lab is to demonstrate how to use FluxCD to automate image updates in a Kubernetes cluster. The lab will use the [AKS Store Demo](https://github.com/Azure-Samples/aks-store-demo) application as an example.

The intended workflow:
1. Modify application code, then commit and push the change to the repo.
2. Create a new release in GitHub which kicks off a release workflow to build and push an updated container image to a GitHub Container Registry.
3. FluxCD detects the new image and updates the image tag in the cluster.
4. FluxCD rolls out the new image to the cluster.

## Bootstrapping FluxCD

In the first sectionI used the AKS extension to install FluxCD. In this section I will use the Flux CLI to install FluxCD. This will enable us to have a bit more control over the installation and gives the ability yo save Flux resources in the repo.

Navigate to the manifest folder and set the following environment variables:

```bash
cd manifests
$ghUser=$(gh api user --jq .login)
 
$ghToken=$(gh auth token) 
```

Now we're ready to bootstrap FluxCD. Run the following command to bootstrap FluxCD. This command will install FluxCD with additional components to enable image automation. It will also generate the Flux resources and commit them to our Git repo in the clusters/dev directory.

```bash
echo $ghToken | flux bootstrap github create --owner=$ghUser --repository=gitops --personal --path=./clusters/dev --branch=main --reconcile --network-policy --components-extra=image-reflector-controller,image-automation-controller
```

After a few minutes, run the following commands and you should see the Flux resources in the repo.
