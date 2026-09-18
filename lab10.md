# Lab: Crossplane XR & XRD — Building `VPCNetwork` Incrementally, One Resource at a Time

> **Prerequisite:** Complete Module 1 and 2 of the base lab (`provider-aws-ec2` installed and healthy, all six Managed Resource CRDs present).
>
> **Crossplane version:** This lab targets **native Crossplane v2** (tested against v2.4.1). It uses the v2 **`Cluster`** scope throughout — there is no Claim object anywhere in this lab, and `LegacyCluster` scope is never used. You apply the composite resource (the XR) directly, exactly as the app team would.
>
> **Why `Cluster` and not `Namespaced`:** v2's `Namespaced` scope is the more common choice when your provider ships namespaced Managed Resource CRDs (typically under a `.m.<group>` API group). At the time of writing, `provider-aws-ec2` doesn't yet ship namespaced variants of `VPC`, `Subnet`, `InternetGateway`, `RouteTable`, `RouteTableAssociation`, or `SecurityGroup` — only the legacy cluster-scoped CRDs exist (`vpcs.ec2.aws.upbound.io`, etc.). Kubernetes doesn't allow a cluster-scoped object to have an owner reference to a namespaced one, so a `Namespaced`-scope XR can never successfully compose these six resources. `Cluster` scope is the other modern, Claim-free v2 option, and it *can* own cluster-scoped resources directly — check `kubectl get crd | grep -i '\.m\.'` against your own provider version before assuming either scope; if namespaced MR CRDs do exist for your provider release, `Namespaced` scope works just as well and only needs `metadata.namespace` added back onto the XR and its `kubectl` commands.

### What v2-native (no Claims) means for this lab

In v1 (and in v2's `LegacyCluster` scope), the app team applies a namespaced **Claim**, and Crossplane creates a separate cluster-scoped **XR** behind it. This lab skips that layer entirely: the XRD is declared with `scope: Cluster`, so the XR itself (`XVPCNetwork`) is a cluster-scoped object, applied directly — there's no claim kind, no `claimRef`, and no second object to keep in sync. `Cluster` scope still uses the modern `spec.crossplane.compositionRef` field (like `Namespaced` does); only `LegacyCluster` uses the old flat `spec.compositionRef` plus Claims.

This lab also does **not** publish a connection Secret at any stage. Every field you need to confirm a resource came up correctly (VPC ID, Subnet ID, etc.) is read straight off the Managed Resource's own `status`, via `kubectl get <mr> -o yaml` or `kubectl describe <mr>`. There's no `connectionSecretKeys` on the XRD, no `connectionDetails` on any composed resource, and no `writeConnectionSecretToRef` anywhere in the Composition.

---


---

## Module 2: Install the Function (once)

Every stage's resource is handled by the same function, so install it before Stage 1 and never touch it again.

```bash
vi function-patch-and-transform.yaml
```

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata:
  name: function-patch-and-transform
spec:
  package: xpkg.crossplane.io/crossplane-contrib/function-patch-and-transform:v0.10.0
```

Check `crossplane.io`'s package registry for the actual current version before installing, since function releases move independently of this lab.

```bash
kubectl apply -f function-patch-and-transform.yaml
kubectl get functions.pkg.crossplane.io
kubectl get pods -n crossplane-system | grep function-patch-and-transform
```

Wait for `INSTALLED: True` and `HEALTHY: True` before Stage 1.

---

## Stage 1: XRD + XR for VPC Only

### 2a. Recap the Managed Resource this stage wraps

```bash
kubectl explain vpc.spec.forProvider
```

This is what you were applying by hand in the previous lab, with no XRD or Composition involved at all — Crossplane's `provider-aws-ec2` talking directly to a `VPC` Managed Resource:

```yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: VPC
metadata:
  name: team-a-vpc
spec:
  forProvider:
    region: us-east-1
    cidrBlock: 10.0.0.0/16
    enableDnsSupport: true
    enableDnsHostnames: true
  providerConfigRef:
    name: default
```


### 2b. XRD — schema exposes only VPC's fields, `scope: Namespaced`

```bash
vi xrd-vpcnetwork.yaml
```

```yaml
apiVersion: apiextensions.crossplane.io/v2
kind: CompositeResourceDefinition
metadata:
  name: xvpcnetworks.aws.platform.example.org
spec:
  scope: Cluster
  group: aws.platform.example.org
  names:
    kind: XVPCNetwork
    plural: xvpcnetworks
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
                      description: AWS region for every resource in the stack
                    vpcCidrBlock:
                      type: string
                      description: CIDR block for the VPC
                  required:
                    - region
                    - vpcCidrBlock
              required:
                - parameters
```

```bash
kubectl apply -f xrd-vpcnetwork.yaml
kubectl describe xrd xvpcnetworks.aws.platform.example.org
```

Confirm `Established: True`. There's no `Offered: True` to check — that status only applies to claim-offering XRDs, and this one never offers a claim.

### 2c. Composition — one pipeline step, one resource

```bash
vi composition-vpcnetwork.yaml
```

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: vpcnetwork-aws
  labels:
    provider: aws
spec:
  compositeTypeRef:
    apiVersion: aws.platform.example.org/v1alpha1
    kind: XVPCNetwork
  mode: Pipeline
  pipeline:
    - step: network
      functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
          - name: vpc
            base:
              apiVersion: ec2.aws.upbound.io/v1beta1
              kind: VPC
              spec:
                forProvider:
                  enableDnsSupport: true
                  enableDnsHostnames: true
                providerConfigRef:
                  name: default
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.region
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.vpcCidrBlock
                toFieldPath: spec.forProvider.cidrBlock
```

```bash
kubectl apply -f composition-vpcnetwork.yaml
kubectl describe composition vpcnetwork-aws
```

Confirm `Synced: True`.

### 2d. XR — apply directly, cluster-scoped, only VPC-shaped parameters

```bash
vi xr-vpcnetwork.yaml
```

```yaml
apiVersion: aws.platform.example.org/v1alpha1
kind: XVPCNetwork
metadata:
  name: team-a-network
spec:
  crossplane:
    compositionRef:
      name: vpcnetwork-aws
  parameters:
    region: us-east-1
    vpcCidrBlock: 10.0.0.0/16
```

**Note on `spec.crossplane`:** in both modern v2 scopes (`Namespaced` and `Cluster`), the XR's `spec` is entirely your own schema — there's no separate Claim object to hold user fields, so Crossplane can't put its own control fields (composition selection, resource refs, etc.) directly under `spec` without risking a collision with something you define. It nests all of that under `spec.crossplane` instead. `compositionRef` therefore lives at `spec.crossplane.compositionRef`, not `spec.compositionRef`.

**Note on scope:** this XRD uses `scope: Cluster`, so the XR has no `metadata.namespace` and no `kubectl get`/`describe` command below needs a `-n <namespace>` flag. If your own provider ships namespaced MR CRDs (check `kubectl get crd | grep -i '\.m\.'`), you'd use `scope: Namespaced` instead, add `namespace: default` back to this manifest, and add `-n default` to every command that follows.

```bash
kubectl apply -f xr-vpcnetwork.yaml
kubectl get xvpcnetwork,vpc
```

### 2e. Verify Stage 1 — one owned resource

```bash
kubectl describe xvpcnetwork team-a-network
```

`Resource Refs` should list exactly **one** entry: the VPC.

```bash
kubectl get vpc -o wide
kubectl get vpc <vpc-resource-name> -o jsonpath='{.status.atProvider.id}'
```

That last command reads the real AWS VPC ID straight off the Managed Resource's own status — this is the pattern you'll reuse at every later stage instead of checking a Secret.


---

## Stage 2: Append Subnet

### 3a. Recap the Managed Resource this stage wraps

```bash
kubectl explain subnet.spec.forProvider
```

New fields needed on the XR: `subnetCidrBlock`, `availabilityZone`.

### 3b. Edit the XRD — add the two new parameters

```bash
vi xrd-vpcnetwork.yaml
```

Add `subnetCidrBlock` and `availabilityZone` under `spec.parameters.properties`, and add both to `required`:

```yaml
                    subnetCidrBlock:            # ← appended
                      type: string
                      description: CIDR block for the public subnet
                    availabilityZone:           # ← appended
                      type: string
                      description: AZ for the subnet
                  required:
                    - region
                    - vpcCidrBlock
                    - subnetCidrBlock           # ← appended
                    - availabilityZone          # ← appended
```

```bash
kubectl apply -f xrd-vpcnetwork.yaml
kubectl explain xvpcnetwork.spec.parameters
```

Confirm the new fields show up on the XR-facing schema.

### 3c. Edit the Composition — append `subnet` directly into the existing step's `resources:` array

The `vpc` entry stays untouched. `subnet` is appended as a **second item in the same `network` step's `resources:` list** — no new step, no new function.

```bash
vi composition-vpcnetwork.yaml
```

```yaml
spec:
  compositeTypeRef:
    apiVersion: aws.platform.example.org/v1alpha1
    kind: XVPCNetwork
  mode: Pipeline
  pipeline:
    - step: network
      functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
          - name: vpc
            base:
              apiVersion: ec2.aws.upbound.io/v1beta1
              kind: VPC
              spec:
                forProvider:
                  enableDnsSupport: true
                  enableDnsHostnames: true
                providerConfigRef:
                  name: default
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.region
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.vpcCidrBlock
                toFieldPath: spec.forProvider.cidrBlock

          # ↓↓↓ new resource, appended into the SAME step's resources list ↓↓↓
          - name: subnet
            base:
              apiVersion: ec2.aws.upbound.io/v1beta1
              kind: Subnet
              spec:
                forProvider:
                  mapPublicIpOnLaunch: true
                  vpcIdSelector:
                    matchControllerRef: true
                providerConfigRef:
                  name: default
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.region
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.subnetCidrBlock
                toFieldPath: spec.forProvider.cidrBlock
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.availabilityZone
                toFieldPath: spec.forProvider.availabilityZone
```

Notice the `pipeline` array itself is still just **one entry** (`step: network`) — only its `resources:` list grew, from one item to two. `vpcIdSelector.matchControllerRef: true` on the Subnet works because the Subnet just needs to be owned by the same XR as the VPC — both entries live in the same step, but that ownership check doesn't care about step or list position.

```bash
kubectl apply -f composition-vpcnetwork.yaml
kubectl describe composition vpcnetwork-aws
```

### 3d. Update the XR with the new parameters, re-apply

```bash
vi xr-vpcnetwork.yaml
```

```yaml
spec:
  crossplane:
    compositionRef:
      name: vpcnetwork-aws
  parameters:
    region: us-east-1
    vpcCidrBlock: 10.0.0.0/16
    subnetCidrBlock: 10.0.1.0/24     # ← appended
    availabilityZone: us-east-1a     # ← appended
```

```bash
kubectl apply -f xr-vpcnetwork.yaml
kubectl get xvpcnetwork,vpc,subnet
```

### 3e. Verify Stage 2 — one more owned resource

```bash
kubectl describe xvpcnetwork team-a-network
```

`Resource Refs` should now list **two** entries: VPC and Subnet. The VPC's ref is unchanged from Stage 1 — appending to the resources list didn't touch it.

```bash
kubectl get subnet -o wide
kubectl get subnet <subnet-resource-name> -o jsonpath='{.status.atProvider.id}'
```

---

## Stage 3: Append InternetGateway

### 4a. Recap the Managed Resource

```bash
kubectl explain internetgateway.spec.forProvider
```

No new XR parameters needed — an Internet Gateway only needs `region` and a VPC to attach to, both already available.

### 4b. XRD — no schema change this stage

Nothing to edit. Not every appended resource needs a new parameter — some resources exist purely to make the network functional and don't add anything to the schema.

### 4c. Composition — append `internet-gateway` as a third entry in the same `resources:` list

Same pattern as Stage 2: no new step, no new function — just a third item appended to the `network` step's `resources:` array, right after `subnet`.

```yaml
          # ↓↓↓ new resource, appended into the SAME step's resources list ↓↓↓
          - name: internet-gateway
            base:
              apiVersion: ec2.aws.upbound.io/v1beta1
              kind: InternetGateway
              spec:
                forProvider:
                  vpcIdSelector:
                    matchControllerRef: true
                providerConfigRef:
                  name: default
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.region
```

```bash
kubectl apply -f composition-vpcnetwork.yaml
```

### 4d. XR — unchanged; re-apply anyway to force reconciliation if needed

```bash
kubectl apply -f xr-vpcnetwork.yaml
kubectl get xvpcnetwork,vpc,subnet,internetgateway
```

### 4e. Verify Stage 3

```bash
kubectl describe xvpcnetwork team-a-network
```

`Resource Refs` now lists **three** entries, all still coming out of the single `network` step.

---

## Stage 4: Append RouteTable

### 5a. Recap the Managed Resource

```bash
kubectl explain routetable.spec.forProvider
```

Needs a VPC to belong to and the Internet Gateway from Stage 3 as its default route's target — both resolved via selectors, no new XR parameters.

### 5b. Composition — append `route-table` as a fourth entry in the same `resources:` list

```yaml
          # ↓↓↓ new resource, appended into the SAME step's resources list ↓↓↓
          - name: route-table
            base:
              apiVersion: ec2.aws.upbound.io/v1beta1
              kind: RouteTable
              spec:
                forProvider:
                  vpcIdSelector:
                    matchControllerRef: true
                  route:
                    - destinationCidrBlock: 0.0.0.0/0
                      gatewayIdSelector:
                        matchControllerRef: true
                providerConfigRef:
                  name: default
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.region
```

`gatewayIdSelector.matchControllerRef: true` here resolves the Internet Gateway the same way the Subnet's `vpcIdSelector` resolved the VPC in Stage 2 — the mechanism is "owned by the same XR," not "listed earlier in the array," so list order here is for readability, not correctness.

```bash
kubectl apply -f composition-vpcnetwork.yaml
kubectl apply -f xr-vpcnetwork.yaml
kubectl get xvpcnetwork,vpc,subnet,internetgateway,routetable
```

### 5c. Verify Stage 4

```bash
kubectl describe xvpcnetwork team-a-network
```

**Four** owned resources now.

---

## Stage 5: Append RouteTableAssociation

### 6a. Recap the Managed Resource

```bash
kubectl explain routetableassociation.spec.forProvider
```

Ties the Subnet (Stage 2) to the RouteTable (Stage 4) — two selectors, still no new XR parameters.

### 6b. Composition — append `route-table-association` as a fifth entry in the same `resources:` list

```yaml
          # ↓↓↓ new resource, appended into the SAME step's resources list ↓↓↓
          - name: route-table-association
            base:
              apiVersion: ec2.aws.upbound.io/v1beta1
              kind: RouteTableAssociation
              spec:
                forProvider:
                  subnetIdSelector:
                    matchControllerRef: true
                  routeTableIdSelector:
                    matchControllerRef: true
                providerConfigRef:
                  name: default
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.region
```

```bash
kubectl apply -f composition-vpcnetwork.yaml
kubectl apply -f xr-vpcnetwork.yaml
kubectl get xvpcnetwork,vpc,subnet,internetgateway,routetable,routetableassociation
```

### 6c. Verify Stage 5

```bash
kubectl describe xvpcnetwork team-a-network
```

**Five** owned resources now.

---

## Stage 6: Append SecurityGroup

### 7a. Recap the Managed Resource

```bash
kubectl explain securitygroup.spec.forProvider
```

Last piece — attaches to the VPC, no new XR parameters.

### 7b. Composition — append `security-group` as the sixth and final entry in the same `resources:` list

```yaml
          # ↓↓↓ new resource, appended into the SAME step's resources list ↓↓↓
          - name: security-group
            base:
              apiVersion: ec2.aws.upbound.io/v1beta1
              kind: SecurityGroup
              spec:
                forProvider:
                  vpcIdSelector:
                    matchControllerRef: true
                  description: Default security group managed via VPCNetwork composition
                providerConfigRef:
                  name: default
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.region
```

```bash
kubectl apply -f composition-vpcnetwork.yaml
kubectl apply -f xr-vpcnetwork.yaml
```

### 7c. Final verify — full chain, one step, six resources

```bash
kubectl get vpc,subnet,internetgateway,routetable,routetableassociation,securitygroup
```

Confirm `READY` and `SYNCED` are `True` across every row.

```bash
kubectl describe xvpcnetwork team-a-network
```

`Resource Refs` now lists all **six** Managed Resources, in the order they were appended.

```bash
kubectl describe composition vpcnetwork-aws
```

`spec.pipeline` still has exactly **one** `step` entry (`network`) — its `resources:` list is the one that grew from one entry to six across the six stages.

**Cross-verify against AWS:**

```bash
aws ec2 describe-vpcs --filters "Name=cidr,Values=10.0.0.0/16"
```

---

## Module 8: Cleanup

Deleting the XR cascades to every composed Managed Resource, regardless of which stage introduced it:

```bash
kubectl delete -f xr-vpcnetwork.yaml
kubectl get vpc,subnet,internetgateway,routetable,routetableassociation,securitygroup
```

Confirm all six are gone, then remove the platform-level objects:

```bash
kubectl delete -f composition-vpcnetwork.yaml
kubectl delete -f xrd-vpcnetwork.yaml
kubectl delete -f function-patch-and-transform.yaml
kubectl api-resources | grep platform.example.org
```

The last command should return nothing once the XRD is gone.

---

