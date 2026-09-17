# Installing Crossplane on Kubernetes with Helm

A step-by-step guide to install Crossplane into your Kubernetes cluster using Helm.

## Prerequisites

- A running Kubernetes cluster.
  If you don't have one, create a local cluster with Kind (Kubernetes IN Docker):

  ```bash
  kind create cluster
  ```

- Helm installed on your machine.
  Install Helm: https://helm.sh/docs/intro/install/

## Step 1: Add the Crossplane Helm Chart Repository

```bash
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
```

## Step 2 (Optional): Dry-Run the Installation

This won't install anything — it just shows what Helm is going to do:

```bash
helm install crossplane \
  crossplane-stable/crossplane \
  --dry-run --debug \
  --namespace crossplane-system \
  --create-namespace
```

## Step 3: Install Crossplane

```bash
helm install crossplane \
  crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace
```

## Step 4: Verify the Pods Are Running

```bash
kubectl get pods -n crossplane-system
```

Expected output:

```
NAME                                      READY   STATUS    RESTARTS   AGE
crossplane-xxxxxxx-xxxxx                  1/1     Running   0          1m
crossplane-rbac-manager-xxxxxxx-xxxxx     1/1     Running   0          1m
```

## Step 5: Verify the Crossplane API Resources (CRDs)

Once the pods are running, Crossplane registers a set of Custom Resource Definitions (CRDs) with the cluster. Confirm they're present:

```bash
kubectl api-resources | grep crossplane
```

Expected output (abridged — actual CRDs vary by Crossplane version):

```
compositeresourcedefinitions   xrd,xrds     apiextensions.crossplane.io/v1   true    CompositeResourceDefinition
compositionrevisions           comprev      apiextensions.crossplane.io/v1   true    CompositionRevision
compositions                   comp         apiextensions.crossplane.io/v1   true    Composition
configurationrevisions                      pkg.crossplane.io/v1             true    ConfigurationRevision
configurations                 configuration pkg.crossplane.io/v1            true    Configuration
controllerconfigs                           pkg.crossplane.io/v1alpha1       true    ControllerConfig
deploymentruntimeconfigs       drconfig     pkg.crossplane.io/v1beta1        true    DeploymentRuntimeConfig
functionrevisions                           pkg.crossplane.io/v1            true    FunctionRevision
functions                      fn           pkg.crossplane.io/v1            true    Function
locks                                       pkg.crossplane.io/v1beta1       true    Lock
providerrevisions                          pkg.crossplane.io/v1            true    ProviderRevision
providers                                  pkg.crossplane.io/v1            true    Provider
storeconfigs                               secrets.crossplane.io/v1alpha1  true    StoreConfig
```

If this list is empty, the Crossplane pods likely haven't finished starting yet — re-check `kubectl get pods -n crossplane-system` and wait a bit before retrying.

## Next Steps

Want help installing a specific cloud provider (like AWS, GCP, or Azure) so Crossplane can manage that provider's resources? Just ask.
