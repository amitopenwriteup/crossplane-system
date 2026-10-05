# Lab: Crossplane XRD & XR Basics (Crossplane v2) — One Resource, One API

Before wrapping six resources into a `VPCNetwork` API, see the XRD/Composition/XR pattern with the smallest possible example: **one Managed Resource, one field.** This lab wraps a single `Bucket` (from `provider-aws-s3`) behind a custom `SimpleBucket` API.

> **Prerequisites:** Crossplane **v2.x** installed, and `provider-aws-s3` **v2.x** installed and healthy. Provider v2 ships *namespaced* managed resources (`*.aws.m.upbound.io`), which this lab uses. Module 0 below checks this and fixes it if needed.

---

## Module 0: Verify the Provider is v2.x (and has a ClusterProviderConfig)

This lab composes **namespaced** Buckets (`s3.aws.m.upbound.io`). A `provider-aws-s3` **v1.x** package only ships the cluster-scoped `s3.aws.upbound.io` API, so the Composition fails later with:

```
cannot compose resources: cannot check if composed resource "bucket" is namespaced ...
no matches for kind "Bucket" in version "s3.aws.m.upbound.io/v1beta1"
```

**Check the installed version and CRDs:**

```bash
kubectl get provider.pkg.crossplane.io
kubectl get crd | grep buckets.s3
```

You need `provider-aws-s3` at `v2.x` and **both** `buckets.s3.aws.upbound.io` (legacy, cluster-scoped) and `buckets.s3.aws.m.upbound.io` (namespaced). If the namespaced CRD is missing, upgrade the provider.

**Upgrade the provider to match the provider family version:**

```bash
vi provider-aws-s3.yaml
```

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-s3
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v2.8.2
```

> Use the same version as `upbound-provider-family-aws` (`kubectl get provider.pkg.crossplane.io`), or the newest v2.x tag on the Upbound Marketplace.

```bash
kubectl apply -f provider-aws-s3.yaml
kubectl get providerrevision.pkg.crossplane.io -w | grep aws-s3
```

A new revision appears as `Active` (the old v1 one becomes `Inactive`). Wait until the new revision shows healthy `True`; this takes a few minutes while the image is pulled and the new CRDs are installed.

```bash
kubectl get pods -n crossplane-system | grep provider-aws-s3
kubectl get crd buckets.s3.aws.m.upbound.io
```

**Create a v2 `ClusterProviderConfig`.** The v1 `ProviderConfig` (group `aws.upbound.io`) is a different API and is not used by namespaced Buckets.

```bash
kubectl get clusterproviderconfig.aws.m.upbound.io
```

If there is no `default`, create one:

```bash
vi clusterproviderconfig.yaml
```

```yaml
apiVersion: aws.m.upbound.io/v1beta1
kind: ClusterProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: creds
```

```bash
kubectl apply -f clusterproviderconfig.yaml
```

Adjust the secret name and key to match your existing AWS credentials secret.

> Existing v1 Buckets from earlier labs are unaffected: a v2 provider keeps serving the legacy cluster-scoped APIs alongside the new ones.

---

## What changed from the v1 lab

| v1 lab | v2 lab |
| --- | --- |
| XRD `apiextensions.crossplane.io/v1` | XRD `apiextensions.crossplane.io/v2` |
| `claimNames` + a Claim you apply | **No Claims.** XRs are namespaced, so you apply the XR directly |
| Cluster-scoped `XSimpleBucket` + namespaced `SimpleBucket` | One namespaced `SimpleBucket` |
| `spec.compositionRef` | `spec.crossplane.compositionRef` (Crossplane machinery lives under `spec.crossplane`) |
| `spec.parameters.region` | `spec.region` (your fields can sit directly under `spec`) |
| `referenceable: true` | removed (not needed in v2) |
| Bucket `s3.aws.upbound.io/v1beta1` (cluster-scoped) | Bucket `s3.aws.m.upbound.io/v1beta1` (namespaced) |
| `providerConfigRef: {name: default}` | `providerConfigRef: {kind: ClusterProviderConfig, name: default}` |
| Composition `apiextensions.crossplane.io/v1`, Pipeline mode | Unchanged (Composition is still `v1`) |

---

## Module 1: The Three Pieces, Minimal Version

| Object | Role here |
| --- | --- |
| **XRD** | Defines a namespaced `SimpleBucket` API with exactly one input: `region` |
| **Composition** | Says "a `SimpleBucket` becomes one `Bucket` Managed Resource" |
| **XR** | The `SimpleBucket` you apply (namespaced) |

```
XR (SimpleBucket, namespaced)
   │
   ▼
Bucket (namespaced Managed Resource from provider-aws-s3)
```

---

## Module 2: Write the XRD

**Explain first:**

```bash
kubectl explain compositeresourcedefinition.spec
```

```bash
vi xrd-simplebucket.yaml
```

```yaml
apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
metadata:
  name: simplebuckets.storage.example.org
spec:
  scope: Namespaced
  group: storage.example.org
  names:
    kind: SimpleBucket
    plural: simplebuckets
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                region:
                  type: string
                  description: AWS region for the bucket
              required:
                - region
```

> `referenceable: true` is harmless in v2 and still accepted; omit it if your CRD schema rejects it.

```bash
kubectl apply -f xrd-simplebucket.yaml
kubectl get xrd
kubectl describe xrd simplebuckets.storage.example.org
```

Look for `Established: True`.

**Confirm the new API exists:**

```bash
kubectl api-resources | grep storage.example.org
kubectl explain simplebucket.spec
```

You should see `region` plus the `crossplane` machinery block Crossplane adds automatically — nothing from the underlying `Bucket` CRD leaks through.

---

## Module 3: Install the Functions

Every Composition is a **Pipeline** of **Functions**. This lab uses two:

- `function-patch-and-transform` — base-and-patch composition (no custom logic).
- `function-auto-ready` — marks the XR `Ready` once all composed resources are ready.

```bash
vi functions.yaml
```

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-patch-and-transform
spec:
  package: xpkg.crossplane.io/crossplane-contrib/function-patch-and-transform:v0.10.3
---
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-auto-ready
spec:
  package: xpkg.crossplane.io/crossplane-contrib/function-auto-ready:v0.6.3
```

> Packages now live under `xpkg.crossplane.io` (previously `xpkg.upbound.io`). Check the [Crossplane docs](https://docs.crossplane.io) for the newest function versions.

```bash
kubectl apply -f functions.yaml
kubectl get functions
```

Confirm `INSTALLED` and `HEALTHY` are both `True` on both before moving on.

```bash
kubectl get pods -n crossplane-system
```

Each Function gets its own pod, the same way a Provider does.

---

## Module 4: Write the Composition (Pipeline Mode)

**Explain first:**

```bash
kubectl explain composition.spec.pipeline
```

```bash
vi composition-simplebucket.yaml
```

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: simplebucket-aws
spec:
  compositeTypeRef:
    apiVersion: storage.example.org/v1alpha1
    kind: SimpleBucket
  mode: Pipeline
  pipeline:
    - step: patch-and-transform
      functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
          - name: bucket
            base:
              apiVersion: s3.aws.m.upbound.io/v1beta1
              kind: Bucket
              spec:
                forProvider: {}
                providerConfigRef:
                  kind: ClusterProviderConfig
                  name: default
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.region
                toFieldPath: spec.forProvider.region
    - step: automatically-detect-ready-composed-resources
      functionRef:
        name: function-auto-ready
```

Key points:

- **`mode: Pipeline`** marks this as a function-based Composition (the only mode in v2).
- **Namespaced Bucket:** the composed `Bucket` is created in the same namespace as the XR automatically — you don't set `metadata.namespace`.
- **`ClusterProviderConfig`:** v2 providers have cluster-wide `ClusterProviderConfig` and namespaced `ProviderConfig`. If your lab uses a namespaced one, change `kind` to `ProviderConfig`.

```bash
kubectl apply -f composition-simplebucket.yaml
kubectl get composition
kubectl describe composition simplebucket-aws
```

---

## Module 5: Create the XR

No Claim step in v2 — you apply the namespaced XR directly.

```bash
vi xr-simplebucket.yaml
```

```yaml
apiVersion: storage.example.org/v1alpha1
kind: SimpleBucket
metadata:
  name: my-first-bucket
  namespace: default
spec:
  region: us-east-1
  crossplane:
    compositionRef:
      name: simplebucket-aws
```

```bash
kubectl apply -f xr-simplebucket.yaml
kubectl get simplebucket -n default
```

**No new pod — same provider pod reconciles it:**

```bash
kubectl get pods -n crossplane-system -l pkg.crossplane.io/provider=provider-aws-s3
```

---

## Module 6: Trace XR → Managed Resource

Use the fully qualified Bucket name to avoid clashing with any legacy cluster-scoped `Bucket`:

```bash
kubectl get simplebucket,buckets.s3.aws.m.upbound.io -n default
```

Confirm `READY` and `SYNCED` are `True` on both.

```bash
kubectl describe simplebucket my-first-bucket -n default
```

Note the `Resource Refs` under `Spec > Crossplane`, pointing at the composed `Bucket`. Then:

```bash
kubectl get buckets.s3.aws.m.upbound.io -n default
kubectl describe buckets.s3.aws.m.upbound.io <name-from-above> -n default
```

You can also render the whole tree:

```bash
crossplane beta trace simplebucket my-first-bucket -n default
```

**Cross-verify against AWS:**

```bash
aws s3api list-buckets --query "Buckets[?contains(Name, 'my-first-bucket')]"
```

---

## Module 7: Cleanup

```bash
kubectl delete -f xr-simplebucket.yaml
kubectl get buckets.s3.aws.m.upbound.io -n default
kubectl delete -f composition-simplebucket.yaml
kubectl delete -f xrd-simplebucket.yaml
kubectl api-resources | grep storage.example.org
```

The last command should return nothing.

> **Leave `functions.yaml` applied.** Both Functions are cluster-wide and reusable — every future Composition (including the `VPCNetwork` lab) references them by name. Remove them with `kubectl delete -f functions.yaml` only if you're tearing down Crossplane entirely.

---

## Troubleshooting

### `no matches for kind "Bucket" in version "s3.aws.m.upbound.io/v1beta1"`

The XR shows this in `kubectl describe simplebucket my-first-bucket -n default` under Events:

```
Warning  ComposeResources  ...  cannot compose resources: cannot check if composed resource "bucket"
is namespaced (a Bucket named ): failed to get restmapping: no matches for kind "Bucket"
in version "s3.aws.m.upbound.io/v1beta1"
```

The namespaced Bucket CRD isn't registered. Work through these in order:

**1. Is the provider on v2.x?**

```bash
kubectl get provider.pkg.crossplane.io
kubectl get providerrevision.pkg.crossplane.io | grep aws-s3
```

- Package `v1.x` → upgrade it (Module 0).
- Package `v2.x` with the new revision `Active` but healthy `False` → it's still starting. Wait and watch with `kubectl get providerrevision.pkg.crossplane.io -w | grep aws-s3`.

**2. Is the new revision failing?**

```bash
kubectl get pods -n crossplane-system | grep provider-aws-s3
kubectl describe providerrevision.pkg.crossplane.io <new-revision-name> | tail -25
kubectl logs -n crossplane-system deploy/<new-revision-name> --tail=30
```

`ImagePullBackOff` or `CrashLoopBackOff` points to a package pull error or a dependency problem with the provider family. If the revision stays `Inactive`, check for `revisionActivationPolicy: Manual` on the Provider.

**3. Does the CRD exist?**

```bash
kubectl get crd buckets.s3.aws.m.upbound.io
```

If it exists but the error persists, Crossplane's REST mapper is stale. Restart it:

```bash
kubectl rollout restart deployment crossplane -n crossplane-system
```

The XR retries automatically, so no reapply is needed once the CRD exists.

**4. Bucket created but not syncing?**

Make sure a `ClusterProviderConfig` named `default` exists (Module 0) and that the Composition's `providerConfigRef` uses `kind: ClusterProviderConfig`.
