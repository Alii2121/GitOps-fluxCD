# GitOps using FluxCD Demo 

### Getting started with flux bootstrap 

``` bash
flux bootstrap github   --owner=$GITHUB_USER   --repository=GitOps-fluxCD   --branch=main   --path=./cluster/   --personal
```

- After bootstrap a namespace is created called `flux-system` with flux controllers deployed inside it and a repo is initialized, `Flux` monitors manifests under `./cluster` path

- Then Created `apps` directory to be used as a base manifest to deploy nginx.

- Each env (Prod,Dev) has a seperate dir and created `kustomize.yaml` file to create overlays specific to each env, In this Demo I have created an overlay with kustomize to deploy 3 replicas in prod namespace and 1 in dev namespace by using the base deployment from apps dir 

- After pushing files to GitHub `flux` takes over and reconciles kustomization to use it as an artifact then deploys any changes to the cluster 

 ### Deploying Helm Chart

- Created a dir for sources such as `HelmRepositories` to create sources yaml manifests 
- Created `HelmCharts` dir and created a release.yaml file to specify `HelmRelease`
- After pushing to git flux took over and deployed the helm chart 

- testing of gitlab trigger 