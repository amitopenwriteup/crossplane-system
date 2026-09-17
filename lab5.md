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

## Module 3: Write the Composition

**Explain first:**

```bash
kubectl explain composition.spec.resources
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
        - fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.region
```

This is the whole mapping: one `base` object, one `patch`. Nothing here references another composed resource, so there's no `matchControllerRef` needed yet — that only shows up once you have more than one resource that must reference each other, as in the VPC lab.

```bash
kubectl apply -f composition-simplebucket.yaml
kubectl get composition
kubectl describe composition simplebucket-aws
```

Look for `Synced: True`.

---

## Module 4: Submit a Claim

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

## Module 5: Trace Claim → XR → Managed Resource

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

## Module 6: Cleanup

```bash
kubectl delete -f claim-simplebucket.yaml
kubectl get bucket
kubectl delete -f composition-simplebucket.yaml
kubectl delete -f xrd-simplebucket.yaml
kubectl api-resources | grep storage.example.org
```

The last command should return nothing.

---

## What's Different in the Full VPCNetwork Lab

Once this pattern feels familiar, the multi-resource `VPCNetwork` lab adds exactly three new ideas on top of what you just did:

1. **Multiple `resources:` entries** in one Composition instead of one.
2. **`matchControllerRef: true`** so composed resources reference each other automatically instead of by name.
3. **`connectionDetails`** to surface values (like a VPC ID) from a composed resource up into a Secret.

Everything else — XRD schema, `compositionRef`, claim → XR → Managed Resource tracing — is identical to what you just did here.
