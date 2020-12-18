# Delivery using [Flux](https://toolkit.fluxcd.io/get-started/)

_NOTE : you have to be into `delivery/flux` folder to run the commands below._

## Install Flux

### Install CLI

```
./setup/install.sh
```

### Install Flux resoures into cluster

Check you cluster :

```
flux check --pre
```

To keep things simple for now, we install Flux in "dev" mode :

```
flux install --arch=amd64

# Note: Flux can also be installed using kubectl
# kubectl apply -f https://github.com/fluxcd/flux2/releases/latest/download/install.yaml
```

_Note: In production, it is recommended to use the `bootstrap` mode to ensure the Flux manifests are also synched with a GIT repository._
_cf. "Bootstrap Flux into the cluster" at the end of this file._

### Check install

```
flux check
```

## Register Git repositories and reconcile them on your cluster

- [Create a git source](https://toolkit.fluxcd.io/cmd/flux_create_source_git/) pointing to a repository main branch:

```
# Replace by your username to point to YOUR fork of the "podtatohead" project
export GITHUB_USER="yogeek" 

flux create source git podtato \
  --url="https://github.com/${GITHUB_USER}/podtato-head" \
  --branch="main" \
  --interval=1m
```

A `GitRepository` Custom Resource has been created:
```
kubectl get gitrepositories.source.toolkit.fluxcd.io -n flux-system
```

- [Create a kustomization](https://toolkit.fluxcd.io/cmd/flux_create_kustomization/) for synchronizing manifests on the cluster:

```
flux create kustomization podtato \
  --source=podtato \
  --path="./delivery/manifest" \
  --prune=true \
  --validation=client \
  --health-check="Deployment/podtatohead.demospace" \
  --health-check-timeout=2m
```

A `Kustomization` Custom Resource has been created:
```
kubectl get kustomizations.kustomize.toolkit.fluxcd.io -n flux-system
```

- In about 30s the synchronization should start:

```
watch flux get kustomizations
```

- When the synchronization finishes you can check that the webapp services are running:

```
kubectl -n demospace get deployments,services
```

You can see you app with its service IP (or with port-forward if you do have a load-balancer):

```
SVC_IP=$(kubectl -n demospace get service podtatohead -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
SVC_PORT=$(kubectl -n demospace get service podtatohead -o jsonpath='{.spec.ports[0].port}')
xdg-open http://${SVC_IP}:${SVC_PORT}
```

From now, any changes made to the Kubernetes manifests in the main branch will be synchronised with the cluster !

- Try to update your source code :

```
# vim ../manifest/manifest.yaml
# Update the image version (reminder: existing tags are 0.1.0, 0.1.1, 0.1.2)

# Commit and push your modification
git add -A && git commit -m "update app" && git push 
```

Observe Flux synching your modifications directly into the cluster:

```
watch flux get kustomizations
```

Refresh you app page in the browser !

**Note**: Even if it is easier to use `flux create` to deploy all Flux objects, the best practice is of course to store the YAML corresponding to these resources in GIT.
To see the manifest for each object, you can use `--export` option or also use the `flux export` command on an existing resource :

```
flux create source git podtato \
  --url="https://github.com/${GITHUB_USER}/podtato-head" \
  --branch="main" \
  --interval=1m \
  --export

flux export source git podtato
```

You can see the resulting files in `./flux-cr/kustomization/` directory.

## Reconciliation details

IMPORTANT :

1. If a Kubernetes manifest is removed from the repository, the reconciler will remove it from your cluster.
2. If you alter the deployment using `kubectl edit`, the changes will be reverted to match the state described in git.
3. If you delete a Kustomization, the reconciler will remove all Kubernetes objects.

## Delete kustomization and source

- Delete the kustomization and see what happens to your resources:

```
flux delete kustomization podtato
```

- Delete the git source :

```
flux delete source git podtato
```

## Register Helm repositories and create Helm releases

https://toolkit.fluxcd.io/guides/helmreleases/

To be able to release a Helm chart, the source that contains the chart (either a HelmRepository, GitRepository, or Bucket) has to be known first to the [source-controller](https://toolkit.fluxcd.io/components/source/controller/), so that the HelmRelease can reference to it.

## Helm release from HelmRepository

```
flux create source helm bitnami \
  --interval=1h \
  --url=https://charts.bitnami.com/bitnami

flux create helmrelease nginx \
  --interval=1h \
  --release-name=nginx \
  --target-namespace=default \
  --source=HelmRepository/bitnami \
  --chart=nginx \
  --chart-version="8.x.x"
```

You now have deployed the `bitnami/nginx` helm chart from the bitnami helm repository.

```
helm ls
```

Check the nginx service :
```
SVC_IP=$(kubectl -n default get service nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')  
SVC_PORT=$(kubectl -n default get service nginx -o jsonpath='{.spec.ports[0].port}')  
xdg-open http://${SVC_IP}:${SVC_PORT}
```

Delete the resources:

```
flux delete helmrelease nginx
flux delete source helm bitnami
```

## Helm release from GitRepository

In our case, the chart is in a Git repo, so we can register a GitRepository source like this :

```
kubectl apply -f ./flux-cr/helm/podtato-chart-gitrepository.yaml
```

And then define a new HelmRelease to release the Helm chart:

```
kubectl apply -f ./flux-cr/helm/podtato-helmrelease.yaml
```

The [helm-controller](https://toolkit.fluxcd.io/components/helm/controller/) will then create a new HelmChart resource in the same namespace as the sourceRef.

You can check the resulting Custom Resources:

```
kubectl get gitrepository.source.toolkit.fluxcd.io,helmrelease.helm.toolkit.fluxcd.io,helmcharts.source.toolkit.fluxcd.io -A
```

Now, you can update the Chart version (in `../charts/podtatohead/Chart.yaml`), push your changes and wait for the reconciliation:

```
# vim ../charts/podtatohead/Chart.yaml

git add -A && git commit -m "update app" && git push

watch flux get helmreleases -A
```

You will see the new Chart version picked up by Flux automatically.

### Bootstrapping

FLux itself should be deployed from a git repository to respect GitOps principles.
To achieve this, Flux provides a `boostrap` command.

Let's delete the "dev-mode" Flux from the cluster and install it in a more "production-ready" workflow.

```
flux uninstall --crds
```

- Check you cluster :

```
flux check --pre
```

- In your github account, create a personnal access token that can create repositories by checking all permissions under repo : https://github.com/settings/tokens

Open another terminal :

```
export GITHUB_USER=<your_username>
export GITHUB_TOKEN=<your_access_token>
export GITHUB_REPO="flux-fleet"
```

- Bootstrap flux with your github repository details :

```
flux bootstrap github \
  --owner="${GITHUB_USER}" \
  --repository=${GITHUB_REPO} \
  --branch=main \
  --path=manifests \
  --personal
```

The bootstrap command creates a repository if one doesn't exist, and commits the manifests for the Flux components to the default branch at the specified path. Then it configures the target cluster to synchronize with the specified path inside the repository.

_Note: The bootstrap command creates a ssh key which it stores as a secret in the Kubernetes cluster. The key is also used to create a deploy key in the GitHub repository. The new deploy key will be linked to the personal access token used to authenticate. Removing the personal access token will remove the deploy key_

- Check installation

```
flux check

kubectl --namespace flux-system get pods
```

- Clone the newly create repository (do not forget to change directory to clone)

```
git clone https://github.com/${GITHUB_USER}/${GITHUB_REPO}.git
cd ${GITHUB_REPO}
tree
```

All Flux manifests are there !

From here, you can commit your applications flux manifests in a `apps` directory of your `flux-fleet` repository:

```
mkdir -p manifests/apps/podtato

flux create source git podtato \
  --url="https://github.com/yogeek/podtato-head" \
  --branch="main" \
  --interval=1m \
  --export | tee manifests/apps/podtato/podtato-app.yaml

flux create kustomization podtato \
  --source=podtato \
  --path="./delivery/manifest" \
  --prune=true \
  --validation=client \
  --health-check="Deployment/podtatohead.demospace" \
  --health-check-timeout=2m \
  --export | tee -a manifests/apps/podtato/podtato-app.yaml

git add .
git commit -m "Added flux configuration for podtato app"
git push -u origin main
```

Watch Flux resources :

```
watch flux get sources git
watch flux get kustomizations
```

Soon, the GitRepository and Kustomization are synched by Flux, and you application is deployed :

```
kubectl get all -n demospace
```

This `flux-fleet` repository can store all your repositories configurations.
Each time you want to add a new application, you just have to add its source configuration in this repository, and Flux will automatically watch it to deploy every modification you will do !

You now have a complete GitOps workflow !

## (Optionnal) More complex example with dependencies : Podinfo app

Stay in the `flux-fleet` repository and declare a new application (coming from a public repository this time) :

- Create a git source pointing to a public repository master branch:

```
mkdir -p manifests/apps/podinfo

flux create source git webapp \
  --url=https://github.com/stefanprodan/podinfo \
  --branch=master \
  --interval=30s \
  --export | tee ./manifests/apps/podinfo/webapp-source.yaml
```

- Create a kustomization for synchronizing the common manifests on the cluster:

```
flux create kustomization webapp-common \
  --source=webapp \
  --path="./deploy/webapp/common" \
  --prune=true \
  --validation=client \
  --interval=1h \
  --export | tee ./manifests/apps/podinfo//webapp-common.yaml
```

- Create a kustomization for the backend service that depends on common:

```
flux create kustomization webapp-backend \
  --depends-on=webapp-common \
  --source=webapp \
  --path="./deploy/webapp/backend" \
  --prune=true \
  --validation=client \
  --interval=10m \
  --health-check="Deployment/backend.webapp" \
  --health-check-timeout=2m \
  --export | tee ./manifests/apps/podinfo//webapp-backend.yaml
```

- Create a kustomization for the frontend service that depends on backend:

```
flux create kustomization webapp-frontend \
  --depends-on=webapp-backend \
  --source=webapp \
  --path="./deploy/webapp/frontend" \
  --prune=true \
  --validation=client \
  --interval=10m \
  --health-check="Deployment/frontend.webapp" \
  --health-check-timeout=2m \
  --export | tee ./manifests/apps/podinfo//webapp-frontend.yaml
```

- Push changes to origin:

```
git add -A && git commit -m "add webapp" && git push
```

Watch Flux resources :

```
watch flux get sources git
watch flux get kustomizations
```

You should see kustomizations resources waiting for their dependencies.
When the reconciliation is finished, the podinfo 3-tier app is deployed :

```
kubectl get all -n webapp
```

Check app in your browser :

```
kubectl -n webapp port-forward svc/frontend 8080:80
```

To delete the podinfo application, you simply have to remove its flux manifests from the `flux-fleet/manifests/apps/`

```
rm -rf manifests/apps/podinfo
git add -A && git commit -m "remove podinfo" && git push
```

The GitRepository, Kustomization and the application will be deleted from your cluster.

```
watch flux get kustomizations
```