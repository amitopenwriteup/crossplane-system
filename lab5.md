# Lab: Crossplane XR & XRD Basics — One Resource, One API

Before wrapping six resources into a `VPCNetwork` API, it's worth seeing the XRD/Composition/XR/Claim pattern with the smallest possible example: **one Managed Resource, one field.** This lab wraps a single `Bucket` (from `provider-aws-s3`) behind a custom `SimpleBucket` API.

> **Prerequisite:** `provider-aws-s3` installed and healthy (`kubectl get provider.pkg.crossplane.io`).

---

## Module 1: The Four Pieces, Minimal Version

| Object | Role here |
|---|---|
| **XRD** | Defines a `SimpleBucket` API with exactly one input: `region` |
| **Composition** | Says "a `SimpleBucket` becomes one `Bucket` Managed Resource" |
| **XR** | The cluster-scoped object Crossplane creates automatically |
| **Claim** | The namespaced `SimpleBucket` you actually apply |

```
Claim (SimpleBucket, namespaced)
   │
   ▼
XR (XSimpleBucket, cluster-scoped)
   │
   ▼
Bucket (the Managed Resource from provider-aws-s3)
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
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xsimplebuckets.storage.example.org
spec:
  group: storage.example.org
  names:
    kind: XSimpleBucket
    plural: xsimplebuckets
  claimNames:
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
                parameters:
                  type: object
                  properties:
                    region:
                      type: string
                      description: AWS region for the bucket
                  required:
                    - region
              required:
                - parameters
```

```bash
kubectl apply -f xrd-simplebucket.yaml
kubectl get xrd
kubectl describe xrd xsimplebuckets.storage.example.org
```

Look for `Established: True` and `Offered: True`.

**Confirm the new API exists:**

```bash
kubectl api-resources | grep storage.example.org
kubectl explain simplebucket.spec.parameters
```

You should see only `region` — exactly what you declared, nothing from the underlying `Bucket` CRD leaking through.

---

## Module 3: Install the Patch-and-Transform Function

Recent Crossplane versions removed the old "Resources mode" Composition (a bare `spec.resources:` list). Every Composition is now a **Pipeline** of one or more **Functions** — small components that tell Crossplane what to compose. For plain base-and-patch composition (no custom logic), the community-maintained `function-patch-and-transform` reproduces the old behavior, so you install it once per cluster and every Composition's pipeline calls it.

> **How to tell you need this:** if `kubectl apply` on a Composition fails with `strict decoding error: unknown field "spec.resources"`, your cluster's Crossplane version requires Pipeline mode — this module is why.

```bash
vi function-patch-and-transform.yaml
```

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-patch-and-transform
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-patch-and-transform:v0.7.0
```

```bash
kubectl apply -f function-patch-and-transform.yaml
kubectl get functions
```

Confirm `INSTALLED` and `HEALTHY` are both `True` before moving on — the Composition in the next module references this Function by name, so it must exist first.

```bash
kubectl get pods -n crossplane-system -l pkg.crossplane.io/function=function-patch-and-transform
```

A Function gets its own pod, the same way a Provider does.

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
    kind: XSimpleBucket
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
              apiVersion: s3.aws.upbound.io/v1beta1
              kind: Bucket
              spec:
                forProvider: {}
                providerConfigRef:
                  name: default
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.region
```

Compared to the old shape, three things changed and nothing else did:

- **`mode: Pipeline`** marks this as a function-based Composition.
- **`pipeline:`** holds one step here (`patch-and-transform`), naming the Function to call via `functionRef`.


```bash
kubectl apply -f composition-simplebucket.yaml
kubectl get composition
kubectl describe composition simplebucket-aws
```

Look for `Synced: True`.

---

## Module 5: Submit a Claim

```bash
vi claim-simplebucket.yaml
```

```yaml
apiVersion: storage.example.org/v1alpha1
kind: SimpleBucket
metadata:
  name: my-first-bucket
  namespace: default
spec:
  parameters:
    region: us-east-1
  compositionRef:
    name: simplebucket-aws
```

```bash
kubectl apply -f claim-simplebucket.yaml
kubectl get simplebucket -n default
```

**No new pod — same provider pod reconciles it:**

```bash
kubectl get pods -n crossplane-system -l pkg.crossplane.io/provider=provider-aws-s3
```

---

## Module 6: Trace Claim → XR → Managed Resource

```bash
kubectl get simplebucket,xsimplebucket,bucket
```

Confirm `READY` and `SYNCED` are `True` on all three.

```bash
kubectl describe simplebucket my-first-bucket -n default
```

Note the `Resource Refs` pointing at the `XSimpleBucket`, then:

```bash
kubectl get xsimplebucket -o wide
kubectl describe xsimplebucket <name-from-above>
```

That describe output points at the actual `Bucket` object, which you can inspect exactly as you did in previous labs:

```bash
kubectl describe bucket <name-from-above>
```

**Cross-verify against AWS:**

```bash
aws s3api list-buckets --query "Buckets[?contains(Name, 'my-first-bucket')]"
```

---

## Module 7: Cleanup

```bash
kubectl delete -f claim-simplebucket.yaml
kubectl get bucket
kubectl delete -f composition-simplebucket.yaml
kubectl delete -f xrd-simplebucket.yaml
kubectl api-resources | grep storage.example.org
```

The last command should return nothing.

> **Leave `function-patch-and-transform` installed.** It's a cluster-wide, reusable Function — every future Composition's pipeline (including the `VPCNetwork` lab) references it by name. Only remove it with `kubectl delete -f function-patch-and-transform.yaml` if you're tearing down Crossplane entirely.

---

