# GitOps using FluxCD Demo 

## Deploying Python Application on a K8S cluster using GitOps approach with FluxCD


## Project Specifications 

- Deploying a python app using Github as a single source of truth and `flux` confirgured to apply any changes on this repo 
- Using `Gitlab` with a local Gitlab `runner` configured on host machine to run CI Pipeline and push new images to `Dockerhub`
  

### Prerequisites 

- Any K8s cluster in this Demo we are using `kind` as a local cluster with 2 worker nodes
- Gitlab Runner configured 
  


### Creating a local `kind` cluster with k8s version 1.28

- Create a kind config yaml file to specify 2 workers and 1 master 

``` yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.28.0@sha256:b7a4cad12c197af3ba43202d3efe03246b3f0793f162afb40a33c923952d5b31
- role: worker
  image: kindest/node:v1.28.0@sha256:b7a4cad12c197af3ba43202d3efe03246b3f0793f162afb40a33c923952d5b31
- role: worker
  image: kindest/node:v1.28.0@sha256:b7a4cad12c197af3ba43202d3efe03246b3f0793f162afb40a33c923952d5b31
```
- Then apply the config file 

``` bash 
sudo kind create cluster --name flux-cluster --config ~/kind-config.yaml
```
- Check if the cluster is running 

``` bash 
kubectl get nodes
```


### Getting started with flux bootstrap 

- export your Github username as a variabl 

```bash 
export GITHUB_USER=<your-user>
```

- Run this command to bootsrap flux and its components into the cluster 

``` bash
flux bootstrap github   --owner=$GITHUB_USER   --repository=GitOps-fluxCD   --components-extra=image-reflector-controller,image-automation-controller --branch=main   --path=./cluster/   --personal
```
`--components-extra` is used to deploy images automation controllers 

- After bootstrap a namespace is created called `flux-system` with flux controllers deployed inside it and a repo is initialized, `Flux` monitors manifests under `./cluster` path

- Then Created `apps` directory to be used as a base manifest to deploy the app.

- Each env (Prod,Dev) has a seperate dir and created `kustomize.yaml` file to create overlays specific to each env, In this Demo I have created an overlay with kustomize to deploy 3 replicas in prod namespace and 1 in dev namespace by using the base deployment from apps dir 

- After pushing files to GitHub `flux` takes over and reconciles kustomization to use it as an artifact then deploys any changes to the cluster 

------------------------------------

 ### Deploying Helm Chart

- Created a dir for sources such as `HelmRepositories` to create sources yaml manifests 
- Created `HelmCharts` dir and created a release.yaml file to specify `HelmRelease`
- After pushing to git flux took over and deployed the helm chart 


------------------------------------

## Structure 

.
├── apps
│   ├── base
│   │   ├── py-app
│   │   │   ├── configmap.yaml
│   │   │   ├── deployment.yaml
│   │   │   └── kustomization.yaml
│   │   └── redis
│   │       ├── deployment.yaml
│   │       ├── kustomization.yaml
│   │       └── svc-redis.yaml
│   ├── Dev
│   │   └── py-app
│   │       └── kustomization.yaml
│   ├── kustomization.yaml
│   └── Prod
│       └── py-app
│           └── kustomization.yaml
├── cluster
│   ├── flux-kustomization.yaml
│   ├── flux-system
│   │   ├── gotk-components.yaml
│   │   ├── gotk-sync.yaml
│   │   └── kustomization.yaml
│   └── tenant
│       └── rbac.yaml
├── infra
│   ├── base
│   │   ├── kustomization.yaml
│   │   └── prometheus-release.yaml
│   ├── kustomization.yaml
│   └── Prod
│       └── kustomization.yaml
├── README.md
└── sources
    ├── docker.yaml
    ├── image-automation.yaml
    ├── image-repo.yaml
    ├── kustomization.yaml
    └── prometheus.yaml

#### Now flux monitors mainly the `./cluster` path mainly as we specified in the bootstrap command so how does flux see changes on other dirs?

- Under ./Cluster dir we created a flux kustomization file called `flux-kustomization.yaml` (we cannot use kustomization.yaml name because it's a different type of kustomization) in this file we used the same git Repository resource we created in bootsrtap but we specified different pathes in order to make flux see other dirs 

- Each dir has a `kustomization` file that point to the subdirectories and inside each sub-dir we created a kustomization file to point to manifests we use 

-----------------------------------------

### Image Automation 


- We have a [Gitlab](https://gitlab.com/Ali2121/Py-App-CICD) repo that has our App code and the pipeline to push images to our DockerHub Registry
- We must create a k8s secret `docker type` to make flux be able to authenticate to our DockerHub private registry and be able to scan and fetch images from there 
- We must create a deploy key in our Github repo with read/write access to make flux push and replace image tags in our k8s maninfests
- We created an `ImageRepository` resource to specify our docker registry 
- Created `ImagePolicy` resource to specify how flux gets the latest image tag as shown in files 
- Created `ImageUpdateAutomation` resource to make flux change and push tags to our yaml files manifests 



------------------------


### The flow 

- Run the Gitlab Pipeline 
- New image tag is pushed to DockerHub
- flux image controller checks the registry every x-minutes specified in `interval` in the resource file  (can create a webhook with docker)  
- flux image Policy states which is the latest image 
- flux changes the deployment yaml manifest with the new tag and push it to GitHub
- flux deploys the new image tag after the reconcilation of the GitRepo 

----------------------------------------------------------------


### Some Important Commands 


- To manually reconcile apps dir 
```bash
flux reconcile ks apps --with-source
```
- To get flux image resources 
```bash
 flux get images all
```
- To manually scan Registry 
``` bash
flux reconcile image repository py-app
```
- To get all image tags that is scanned by flux 
``` bash
kubectl -n flux-system describe imagerepositories py-app
```
- To get all flux resources and its status
``` bash
flux get all
``` 
- Wait for Flux to apply the latest commit on the cluster and verify that podinfo was updated
``` bash
watch "kubectl get deployment/podinfo -oyaml | grep 'image:'"
```
 

### Reference 
[Flux-Docs](https://fluxcd.io/flux/get-started/)
  

  
