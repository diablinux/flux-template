# Crossplane the GitOps way

## Overview

Crossplane is an open source Kubernetes extension that transforms your Kubernetes cluster into a universal *control plane*.

https://docs.crossplane.io/latest/

Add Crossplane pod arguments to the `spec.template.spec.containers[].args` section of the `crossplane-values.yaml` file

e.g. you want to enable feature flags like `- --enable-realtime-compositions` option (alpha)

Add the flag:

```bash
vim k8s-addons/crossplane-system/base/helm-release.yaml
```


```yaml
  values:
    # Optional customizations
    args:
      - "--debug" # Enable debug logging
      - "--enable-realtime-compositions"
```

Save the file, commit and push.

```bash
➜  flux-minikube git:(main) ✗ git commit -m "Enable real time compositions support" k8s-addons/crossplane-system/base/helm-release.yaml
[main f21b39c] Enable real time compositions support
 1 file changed, 2 insertions(+), 1 deletion(-)
➜  flux-minikube git:(main) git push
Enumerating objects: 11, done.
Counting objects: 100% (11/11), done.
Delta compression using up to 10 threads
Compressing objects: 100% (4/4), done.
Writing objects: 100% (6/6), 523 bytes | 523.00 KiB/s, done.
Total 6 (delta 2), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
To github.com:diablinux/flux-minikube.git
   e38472e..f21b39c  main -> main
```

Get the pods:

```bash
kubectl get pods -n crossplane-system --watch
NAME                                       READY   STATUS      RESTARTS   AGE
crossplane-6d7fb6d6fc-hcnsr                0/1     Completed   0          19h
crossplane-79fb578876-hs29v                1/1     Running     0          5s
crossplane-rbac-manager-59d8fcb968-qcvvb   1/1     Running     0          19h
crossplane-6d7fb6d6fc-hcnsr                0/1     Completed   0          19h
crossplane-6d7fb6d6fc-hcnsr                0/1     Completed   0          19h
```

Validate the change

```bash
➜  flux-minikube git:(main) k get po crossplane-79fb578876-hs29v -n crossplane-system -o "jsonpath={.spec.containers..args}"
```

```json
["core","start","--debug","--enable-realtime-compositions"]
```

## Install a Provider

Crossplane supports installing Providers during an initial Crossplane installation with the Crossplane Helm chart.

First, crossplane needs a `ProviderConfig`, let's create the `k8s-addons/crossplane-system/base/config-in-cluster.yaml` :

```yaml
# Check ./provider-in-cluster.yaml to see how to grant permissions to the Provider
apiVersion: kubernetes.crossplane.io/v1alpha1
kind: ProviderConfig
metadata:
  name: kubernetes-provider
spec:
  credentials:
    source: InjectedIdentity
```

Let's add the kubernetes provider to crossplane

The file `k8s-addons/crossplane-system/base/provider-in-cluster.yaml` has the `Provider`, `DeploymentRuntimeConfig` and `ClusterRoleBinding` definitions:

```yaml
---
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-kubernetes
spec:
  package: xpkg.upbound.io/upbound/provider-kubernetes:v0.17.0
  runtimeConfigRef:
    apiVersion: pkg.crossplane.io/v1beta1
    kind: DeploymentRuntimeConfig
    name: provider-kubernetes
---
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: provider-kubernetes
spec:
  serviceAccountTemplate:
    metadata:
      name: provider-kubernetes
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: provider-kubernetes-cluster-admin
subjects:
  - kind: ServiceAccount
    name: provider-kubernetes
    namespace: crossplane-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

Track the file with git, commit and push:

```bash
➜  flux-minikube git:(main) ✗ git status
On branch main
Your branch is up to date with 'origin/main'.

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	k8s-addons/crossplane-system/base/provider-in-cluster.yaml

nothing added to commit but untracked files present (use "git add" to track)

➜  flux-minikube git:(main) ✗ git add k8s-addons/crossplane-system/base/provider-in-cluster.yaml

➜  flux-minikube git:(main) ✗ git commit -m "Adding kuberneter provider to crossplane" k8s-addons/crossplane-system/base/provider-in-cluster.yaml
[main db84cb5] Adding kuberneter provider to crossplane
 1 file changed, 33 insertions(+)
 create mode 100644 k8s-addons/crossplane-system/base/provider-in-cluster.yaml

➜  flux-minikube git:(main) git push
Enumerating objects: 10, done.
Counting objects: 100% (10/10), done.
Delta compression using up to 10 threads
Compressing objects: 100% (4/4), done.
Writing objects: 100% (6/6), 782 bytes | 782.00 KiB/s, done.
Total 6 (delta 1), reused 0 (delta 0), pack-reused 0
remote: Resolving deltas: 100% (1/1), completed with 1 local object.
To github.com:diablinux/flux-minikube.git
   b27ac37..db84cb5  main -> main
```

The kubernetes provider pod will be created as soon as the reconciliation process has been triggered.

```shell
➜  flux-minikube git:(main) k get po -n crossplane-system
NAME                                                READY   STATUS    RESTARTS   AGE
crossplane-79fb578876-hs29v                         1/1     Running   0          78m
crossplane-rbac-manager-59d8fcb968-qcvvb            1/1     Running   0          21h
provider-kubernetes-4ac7e128452c-5cb7db99d4-v5f4z   1/1     Running   0          2m5s
```

At flux level we could see:

```bash
➜  flux-minikube git:(main) flux events -A --watch
flux-system	11s	Normal	Progressing	Kustomization/flux-system	
              ClusterRoleBinding/provider-kubernetes-cluster-admin created
           	   	       DeploymentRuntimeConfig/provider-kubernetes created
           	   	                     	Provider/provider-kubernetes created
flux-system	11s	Normal	ReconciliationSucceeded	Kustomization/flux-system	Reconciliation finished in 900.193834ms, next run in 10m0s
```

The kubernetes Provider is running, now it's time to create some kubernetes resources 🚀