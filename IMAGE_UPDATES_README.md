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

We also need to specify the Kustomization resource to tell FluxCD where to find the app deployment manifests in our repo. Run the following command to create the configuration and export it to a YAML file which we'll also commit to our repo.

```bash
flux create kustomization aks-store-demo --source=aks-store-demo --path="./overlays/dev" --prune=true --wait=true --interval=1m --retry-interval=2m --health-check-timeout=3m --export > ./clusters/dev/aks-store-demo-kustomization.yaml
```

Now there are two new Flux resource manifests files. Commit and push the changes to the repo and merge the pull request, so flux can begin the reconciliation process.

Watch after the pull reqeust is merged to see the reconciliation process.

```bash
flux get kustomizations --watch
```

Confirm the GitRepository and Kustomization resources have been created.

```bash
flux get sources git
flux get kustomizations
```
Run the following command to ensure all the Pods are running.

```bash
kubectl get pods -n dev
```

Once all the Pods are running, run the following command to grab the public IP of the store-front service.

```bash
kubectl get svc/store-front -n dev
```

Navigate to the IP address in a browser and you should see the store front page.

## Setup the release workflow

The source code for the `aks-store-demo`` application is located in a different repository than where our manifests are.

Warning: To give you a bit of a warning, we'll be flipping back and forth between the aks-store-demo and aks-store-demo-manifests repository directories.

Hint: you can use the cd - command to flip back and forth between the last two directories you were in.

The application code is in the [aks-store-demo repository](https://github.com/azure-samples/aks-store-demo). Run the following commands to fork and clone the repo.

```bash
# navigate to the home directory or wherever you want to clone the repo
cd -

# fork and clone the repo to your local machine
gh repo fork https://github.com/azure-samples/aks-store-demo.git --clone

# make sure you are in the aks-store-demo directory
cd aks-store-demo

# since we are in a forked repo, we need to set the default to be our fork
gh repo set-default
```

Currently the `aks-store-demo` repository has Continuous Integration (CI) workflows to build and push container images using `GITHUB_SHA` as the image tag. We need to create a new release workflow that will build and push a new container image using [semantic versioning](https://semver.org/). Flux's image update automation policy which is used to determine the most recent image tag can use various methods including numerical tags, alphabetical tags, and semver tags. So in this case, as much as I'd like to use the `GITHUB_SHA`, it does not provide a conducive way to determine "latest".

Therefore, we need to tag our releases with semantic version numbers. GitHub offers a way to create [tagged releases](https://docs.github.com/repositories/releasing-projects-on-github/about-releases). Additionally, GitHub Actions have the ability to [trigger workflows when a new release is published](https://docs.github.com/actions/using-workflows/events-that-trigger-workflows#release).

In the aks-store-demo repository, create a new file called `release-store-front.yaml` in the `.github/workflows` directory. Copy and paste the following contents into the file.

```yaml
name: release-store-front

on:
  release:
    types: [published]

permissions:
  contents: read
  packages: write

jobs:
  publish-container-image:

    runs-on: ubuntu-latest

    steps:
      - name: Set environment variables
        id: set-variables
        run: |
          echo "REPOSITORY=ghcr.io/$(echo ${{ github.repository }} | tr '[:upper:]' '[:lower:]')" >> "$GITHUB_OUTPUT"
          echo "IMAGE=store-front" >> "$GITHUB_OUTPUT"
          echo "VERSION=$(echo ${GITHUB_REF#refs/tags/})" >> "$GITHUB_OUTPUT"
          echo "CREATED=$(date -u +'%Y-%m-%dT%H:%M:%SZ')" >> "$GITHUB_OUTPUT"

      - name: Env variable output
        id: test-variables
        run: |
          echo ${{ steps.set-variables.outputs.REPOSITORY }}
          echo ${{ steps.set-variables.outputs.IMAGE }}
          echo ${{ steps.set-variables.outputs.VERSION }}
          echo ${{ steps.set-variables.outputs.CREATED }}

      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v1
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ github.token }}

      - name: Build and push
        uses: docker/build-push-action@v2
        with:
          context: src/store-front
          file: src/store-front/Dockerfile
          push: true
          tags: |
            ${{ steps.set-variables.outputs.REPOSITORY }}/${{ steps.set-variables.outputs.IMAGE }}:latest
            ${{ steps.set-variables.outputs.REPOSITORY }}/${{ steps.set-variables.outputs.IMAGE }}:${{ steps.set-variables.outputs.VERSION }}
          labels: |
            org.opencontainers.image.source=${{ github.repositoryUrl }}
            org.opencontainers.image.created=${{ steps.set-variables.outputs.CREATED }}
            org.opencontainers.image.revision=${{ steps.set-variables.outputs.VERSION }}
```

This workflow will be triggered each time a new release is published and the `${GITHUB_REF#refs/tags/}` bit allows us to extract the release version to use as an image tag.

Save the file, then commit and push the changes to the repo.

## Update the application

Since we're in the app repo, let's make a change to it.

We'll make a very small edit to the `src/store-front/src/components/TopNav.vue` file and update the title of the application to include a version number.

Open the file and change the title on line 4 from Azure Pet Supplies to Azure Pet Supplies v1.0.0.

Commit and push the changes to the repo.

Using GitHub CLI, create and publish a new release tagged as 1.0.0. This will trigger the release workflow we just created.

```bash
gh release create 1.0.0 --generate-notes
```

Wait about 10 seconds then run the following command to watch the workflow run.

```bash
gh run watch
```

With the `store-front` container image release workflow setup, let's configure image update automation in Flux.

## Configure image update automation

Note: Flip back over to the aks-store-demo-manifests repo!

To setup FluxCD to listen for changes in the GitHub Container Registry, we need to create a new `ImageRepository` resource. You need to create an `ImageRepository` resource for each image you want to automate. In this case, we're only automating the `store-front` image.

Using the FluxCLI, run the following command to create the manifest for the `ImageRepository` resource.

```bash
flux create image repository store-front --image=ghcr.io/$ghUser/aks-store-demo/store-front --interval=1m --export > ./clusters/dev/aks-store-demo-store-front-image.yaml
```

Run the following command to create an `ImagePolicy` resource to tell FluxCD how to determine the newest image tags. We'll use the `semver` filter to only allow image tags that are valid semantic versions and equal to or greater than 1.0.0. There are other [filters](https://fluxcd.io/flux/guides/image-update/#imagepolicy-examples) you can use as well.

```bash
flux create image policy store-front --image-ref=store-front --select-semver='>=1.0.0' --export > ./clusters/dev/aks-store-demo-store-front-image-policy.yaml
```

Finally, run the following command to create an `ImageUpdateAutomation` resource which enables FluxCD to update images tags in our YAML manifests.

```bash
flux create image update store-front --interval=1m --git-repo-ref=aks-store-demo --git-repo-path="./manifest" --checkout-branch=main --author-name=fluxcdbot --author-email=fluxcdbot@users.noreply.github.com --commit-template="{{range .Updated.Images}}{{println .}}{{end}}" --export > ./clusters/dev/aks-store-demo-store-front-image-update.yaml
```

If you are uncomfortable with FluxCD making changes directly to the checkout branch (in this case main), you can create a separate branch for FluxCD (using the --push-branch parameter) to specify where commits should be pushed to. This will then enable you to follow your normal Git workflows (e.g., create a pull request to merge the changes into the main branch).

Commit and push the new Flux resources to the repo.

After a few minutes, run the following command to see the status of the image update automation resources.

```bash
flux get image repository store-front
flux get image policy store-front
flux get image update store-front
```

The image update automation is setup but it won't do anything until we tell it which Deployments to update.

## Update the manifest

We're going to stick with updating the `store-front` only. So we'll need to update the `store-front` deployment manifest to use the `ImagePolicy` resource we created earlier. This is done by marking the manifest with a comment.

Open the `manifest/store-front.yaml` file using your favorite editor.

On line 19, update the image to use your GitHub Container Registry. Then at the end of the line, add a comment to include the namespace and name of your `ImagePolicy` resource. This marks the deployment for Flux to update.

The comment is super important because Flux will not implement the image tag policy if it doesn't have this comment.

The line should look something like this:

```yaml
image: ghcr.io/<REPLACE_THIS_WITH_YOUR_GITHUB_USERNAME>/aks-store-demo/store-front:latest # {"$imagepolicy": "flux-system:store-front"}
```

Setting the marker in the deployment manifest is fine for this demo, but ideally you'd want to set it in the kustomization manifest instead.

Commit and push the new Flux resources to the repo.

At this point, we've successfully setup image automation in our cluster. Time to test.

## Test the image automation

We're going to test the developer workflow we laid out at the beginning of this article.

### Make a change

Flip back over to the aks-store-demo repo

Make another change to the TopNav.vue file. This time, change the title to Azure Pet Supplies v2.0.0.

Commit and push the changes to the repo.

Create a new 2.0.0 release. This will trigger the release workflow.

```bash
gh release create 2.0.0 --generate-notes

# wait about 5 seconds then run the following command
gh run watch
```

## Verify the image update

After a few minutes, you should see the new image tag in the aks-store-demo repo and the store-front deployment being updated in the cluster.

The reconcile interval for ImagePolicy was set to 1 minute. So after 1 minute or so, we should see that the update was successful.

```bash
flux get image policy store-front
```

Now check the store-front deployment to see the new image tag.

```bash
kubectl get deployment store-front -n dev -o=jsonpath='{.spec.template.spec.containers[0].image}'
```

You should see that the image tag has been updated to `2.0.0`.

Get the public IP of the store-front service by running the following command.

```bash
kubectl get svc/store-front -n dev
```

Navigate to the IP address in a browser and you should see the store front page with the new title.