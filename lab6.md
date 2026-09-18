# Lab: Crossplane XR & XRD — Building `VPCNetwork` Incrementally, One Resource at a Time


> **Prerequisite:** Complete Module 1 and 2 of the base lab (`provider-aws-ec2` installed and healthy, all six Managed Resource CRDs present).
>


---

## Module 1: Concepts Recap

| Object | What it is | Grows how, in this lab |
|---|---|---|
| **XRD** | Schema of the `VPCNetwork` API | `spec.parameters` gains a field, `spec.connectionSecretKeys` gains a key, each stage |
| **Function** | Installed package the pipeline calls (`function-patch-and-transform`) | Installed once, in Stage 1, never touched again |
| **Composition** | A `spec.pipeline` array of steps | Gains one new step per stage |
| **XR** (`XVPCNetwork`) | Cluster-scoped composite Crossplane creates from the claim | `status` / owned-resource count grows each stage |
| **Claim** (`VPCNetwork`) | Namespaced object the app team applies | Gains one new parameter per stage, same object re-applied |

```
Stage 1        Stage 2                Stage 3                     ...        Stage 6
pipeline:      pipeline:              pipeline:                              pipeline:
  - vpc           - vpc                  - vpc                                  - vpc
                  - subnet               - subnet                               - subnet
                                         - internet-gateway                     - internet-gateway
                                                                                 - route-table
                                                                                 - route-table-association
                                                                                 - security-group
```

Each row above is a full, valid Composition on its own — you could stop after any stage and have a working (if smaller) API.

### Note: why one step per resource, instead of one step with six resources?

Every step in this lab calls the *same* function (`function-patch-and-transform`), so nothing stops you from putting all six Managed Resources inside a single step's `resources:` array. It would compose the same six MRs, with the same owner refs and the same connection secret keys. Splitting them into separate steps is a design choice, not a Crossplane requirement, made for four reasons:

1. **Per-step status and debugging.** Crossplane reports pipeline execution results per `step` name. With separate steps, a failure tells you exactly which one broke (`step: subnet` failed). With everything in one step's resource list, you only learn "the step failed" and have to scan the whole list to find the culprit.
2. **This lab is deliberately incremental.** The exercise is built around "append one step per stage, watch `Resource Refs` grow by one, watch the connection secret grow." That maps cleanly onto `pipeline: [step1] → [step1, step2] → ...`. Editing an array inside one giant step would give the same end state but a much less obvious diff at each stage.
3. **Steps matter once you mix functions.** Here every step happens to use the same function, so combining them costs nothing functionally. But separate steps become load-bearing the moment a *different* function enters the pipeline — e.g. `function-patch-and-transform` for the six MRs, followed by a step running `function-auto-ready` or `function-extra-resources` that consumes the previous step's output via the pipeline context. That only works with distinct steps, since a step is the unit that's bound to one function.
4. **Readability at scale.** Six resources in one array vs. six named steps is a wash. Twenty or more resources in a single step's array is a much harder file to scan or diff in a PR than twenty named steps.

Keep this in mind as you go through Stages 2–6 below: each stage appends a **step**, but you could just as validly append an entry to the **first step's `resources:` array** and get an identical `XVPCNetwork`. The lab uses steps so that each stage's change is a clean, isolated diff you can point to.

---

## Module 2: Install the Function (once)

Every stage's pipeline step calls the same function, so install it before Stage 1 and never touch it again.

```bash
vi function-patch-and-transform.yaml
```

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: Function
metadata:
  name: function-patch-and-transform
spec:
  package: xpkg.upbound.io/crossplane-contrib/function-patch-and-transform:v0.7.0
```

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

Field by field, this is the vocabulary the XRD and Composition below will be built from:

| Field | Meaning | What happens to it once wrapped |
|---|---|---|
| `apiVersion` / `kind` | Identifies this as `provider-aws-ec2`'s `VPC` CRD | Copied verbatim into the Composition's `base` — the claim never sees it |
| `metadata.name` | Name of *this one* VPC object | Generated by Crossplane for the composed resource; the claim only names the `VPCNetwork`, not each MR inside it |
| `spec.forProvider.region` | AWS region to create the VPC in | Becomes `spec.parameters.region` on the claim — different per claim, so it's a **patched** field |
| `spec.forProvider.cidrBlock` | The VPC's IP range | Becomes `spec.parameters.vpcCidrBlock` on the claim — also patched, renamed so it doesn't collide with the Subnet's own `cidrBlock` later |
| `spec.forProvider.enableDnsSupport` / `enableDnsHostnames` | DNS behavior inside the VPC | Same for every VPC this platform creates, so it's **hardcoded in the Composition's `base`** and never exposed as a parameter at all |
| `spec.providerConfigRef.name` | Which AWS credentials to use | Same for every claim in this cluster, so it's also hardcoded in the `base` |
| `status.atProvider.id` (populated after creation, not shown above) | The real AWS VPC ID (`vpc-0abc...`) | Surfaced via `connectionDetails` into the connection secret's `vpcId` key |

That last column is the whole point of Module 3 onward: every field either becomes a claim parameter (something the app team can vary), stays hardcoded in the Composition (something the platform team fixes), or gets read back out of `status` into a connection secret (something the app team needs after creation). Nothing in the XRD or Composition below introduces a field that isn't already sitting somewhere in this raw MR.

### 2b. XRD — schema exposes only VPC's fields

```bash
vi xrd-vpcnetwork.yaml
```

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xvpcnetworks.aws.platform.example.org
spec:
  group: aws.platform.example.org
  names:
    kind: XVPCNetwork
    plural: xvpcnetworks
  claimNames:
    kind: VPCNetwork
    plural: vpcnetworks
  connectionSecretKeys:
    - vpcId
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
            status:
              type: object
              properties:
                vpcId:
                  type: string
```

```bash
kubectl apply -f xrd-vpcnetwork.yaml
kubectl describe xrd xvpcnetworks.aws.platform.example.org
```

Confirm `Established: True`, `Offered: True`.

### 2c. Composition — pipeline with exactly one step

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
    - step: vpc
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
            connectionDetails:
              - name: vpcId
                type: FromFieldPath
                fromFieldPath: status.atProvider.id
```

```bash
kubectl apply -f composition-vpcnetwork.yaml
kubectl describe composition vpcnetwork-aws
```

Confirm `Synced: True`.

### 2d. Claim — only VPC-shaped parameters

```bash
vi claim-vpcnetwork.yaml
```

```yaml
apiVersion: aws.platform.example.org/v1alpha1
kind: VPCNetwork
metadata:
  name: team-a-network
  namespace: default
spec:
  parameters:
    region: us-east-1
    vpcCidrBlock: 10.0.0.0/16
  compositionRef:
    name: vpcnetwork-aws
  writeConnectionSecretToRef:
    name: team-a-network-conn
```

```bash
kubectl apply -f claim-vpcnetwork.yaml
kubectl get vpcnetwork,xvpcnetwork,vpc
```

### 2e. Verify Stage 1 — one owned resource, one connection key

```bash
kubectl describe xvpcnetwork <name-from-above>
```

`Resource Refs` should list exactly **one** entry: the VPC.

```bash
kubectl get secret team-a-network-conn -n default -o jsonpath='{.data}' | jq 'keys'
```

Should print `["vpcId"]` only — this is the baseline you'll watch grow.

```bash
kubectl get secret team-a-network-conn -n default -o jsonpath='{.data.vpcId}' | base64 -d
```

---

## Stage 2: Append Subnet

### 3a. Recap the Managed Resource this stage wraps

```bash
kubectl explain subnet.spec.forProvider
```

New fields needed on the claim: `subnetCidrBlock`, `availabilityZone`.

### 3b. Edit the XRD — add the two new parameters and the new connection key

```bash
vi xrd-vpcnetwork.yaml
```

Add `subnetCidrBlock` and `availabilityZone` under `spec.parameters.properties`, add both to `required`, and append `subnetId` to `connectionSecretKeys`:

```yaml
  connectionSecretKeys:
    - vpcId
    - subnetId          # ← appended
```

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

And under `status.properties`:

```yaml
                subnetId:                       # ← appended
                  type: string
```

```bash
kubectl apply -f xrd-vpcnetwork.yaml
kubectl explain vpcnetwork.spec.parameters
```

Confirm the new fields show up on the claim-facing schema.

### 3c. Edit the Composition — append a new pipeline step

This is the append the lab is built around: the whole VPC step stays untouched; a second step is added to the `pipeline` array. (As the note in Module 1 explains, you could instead add `subnet` to the *first* step's `resources:` array with identical results — the separate-step structure here is for a clean, isolated diff per stage, not a functional requirement.)

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
    - step: vpc
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
            connectionDetails:
              - name: vpcId
                type: FromFieldPath
                fromFieldPath: status.atProvider.id

    # ↓↓↓ new step, appended ↓↓↓
    - step: subnet
      functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
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
            connectionDetails:
              - name: subnetId
                type: FromFieldPath
                fromFieldPath: status.atProvider.id
```

Note `vpcIdSelector.matchControllerRef: true` in the new step: the Subnet doesn't need to know the VPC's name or which step created it — it only needs to be owned by the same XR, which it is, since both steps compose resources for the one XR that's currently reconciling.

```bash
kubectl apply -f composition-vpcnetwork.yaml
kubectl describe composition vpcnetwork-aws
```

### 3d. Update the claim with the new parameters, re-apply

```bash
vi claim-vpcnetwork.yaml
```

```yaml
spec:
  parameters:
    region: us-east-1
    vpcCidrBlock: 10.0.0.0/16
    subnetCidrBlock: 10.0.1.0/24     # ← appended
    availabilityZone: us-east-1a     # ← appended
  compositionRef:
    name: vpcnetwork-aws
  writeConnectionSecretToRef:
    name: team-a-network-conn
```

```bash
kubectl apply -f claim-vpcnetwork.yaml
kubectl get vpcnetwork,xvpcnetwork,vpc,subnet
```

### 3e. Verify Stage 2 — one more owned resource, one more connection key

```bash
kubectl describe xvpcnetwork <name-from-above>
```

`Resource Refs` should now list **two** entries: VPC and Subnet. The VPC's ref is unchanged from Stage 1 — appending a step didn't touch it.

```bash
kubectl get secret team-a-network-conn -n default -o jsonpath='{.data}' | jq 'keys'
```

Should now print `["subnetId","vpcId"]`.

```bash
kubectl get secret team-a-network-conn -n default -o jsonpath='{.data.subnetId}' | base64 -d
```

---

## Stage 3: Append InternetGateway

### 4a. Recap the Managed Resource

```bash
kubectl explain internetgateway.spec.forProvider
```

No new claim parameters needed — an Internet Gateway only needs `region` and a VPC to attach to, both already available.

### 4b. XRD — no schema change this stage

Nothing to edit. This stage is a good moment to notice that **not every appended resource needs a new parameter or a new connection key** — some resources exist purely to make the network functional and don't surface anything to the caller.

### 4c. Composition — append the third pipeline step

```yaml
    # ↓↓↓ new step, appended after "subnet" ↓↓↓
    - step: internet-gateway
      functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
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

### 4d. Claim — unchanged; re-apply anyway to force reconciliation if needed

```bash
kubectl apply -f claim-vpcnetwork.yaml
kubectl get vpcnetwork,xvpcnetwork,vpc,subnet,internetgateway
```

### 4e. Verify Stage 3

```bash
kubectl describe xvpcnetwork <name-from-above>
```

`Resource Refs` now lists **three** entries. Connection secret keys are unchanged — confirm:

```bash
kubectl get secret team-a-network-conn -n default -o jsonpath='{.data}' | jq 'keys'
```

Still `["subnetId","vpcId"]` — appending a step doesn't force a connection key to appear unless that step's resource declares `connectionDetails`.

---

## Stage 4: Append RouteTable

### 5a. Recap the Managed Resource

```bash
kubectl explain routetable.spec.forProvider
```

Needs a VPC to belong to and the Internet Gateway from Stage 3 as its default route's target — both resolved via selectors, no new claim parameters.

### 5b. Composition — append the fourth step

```yaml
    # ↓↓↓ new step, appended after "internet-gateway" ↓↓↓
    - step: route-table
      functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
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

`gatewayIdSelector.matchControllerRef: true` here is doing the same cross-step resolution as the Subnet's `vpcIdSelector` in Stage 2 — this step is written after the `internet-gateway` step in the array, but the mechanism that finds the gateway is "owned by the same XR," not "produced by the previous step," so step *order* here is for readability, not correctness.

```bash
kubectl apply -f composition-vpcnetwork.yaml
kubectl apply -f claim-vpcnetwork.yaml
kubectl get vpcnetwork,xvpcnetwork,vpc,subnet,internetgateway,routetable
```

### 5c. Verify Stage 4

```bash
kubectl describe xvpcnetwork <name-from-above>
```

**Four** owned resources now.

---

## Stage 5: Append RouteTableAssociation

### 6a. Recap the Managed Resource

```bash
kubectl explain routetableassociation.spec.forProvider
```

Ties the Subnet (Stage 2) to the RouteTable (Stage 4) — two selectors, still no new claim parameters.

### 6b. Composition — append the fifth step

```yaml
    # ↓↓↓ new step, appended after "route-table" ↓↓↓
    - step: route-table-association
      functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
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
kubectl apply -f claim-vpcnetwork.yaml
kubectl get vpcnetwork,xvpcnetwork,vpc,subnet,internetgateway,routetable,routetableassociation
```

### 6c. Verify Stage 5

```bash
kubectl describe xvpcnetwork <name-from-above>
```

**Five** owned resources now.

---

## Stage 6: Append SecurityGroup

### 7a. Recap the Managed Resource

```bash
kubectl explain securitygroup.spec.forProvider
```

Last piece — attaches to the VPC, no new claim parameters.

### 7b. Composition — append the sixth and final step

```yaml
    # ↓↓↓ new step, appended after "route-table-association" ↓↓↓
    - step: security-group
      functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
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
kubectl apply -f claim-vpcnetwork.yaml
```

### 7c. Final verify — full chain, all six steps, both connection keys

```bash
kubectl get vpcnetwork,xvpcnetwork,vpc,subnet,internetgateway,routetable,routetableassociation,securitygroup
```

Confirm `READY` and `SYNCED` are `True` across every row.

```bash
kubectl describe xvpcnetwork <name-from-above>
```

`Resource Refs` now lists all **six** Managed Resources — exactly the six pipeline steps in `composition-vpcnetwork.yaml`, in the order they were appended.

```bash
kubectl describe composition vpcnetwork-aws
```

`spec.pipeline` now has six `step` entries — you can literally count them against the six-resource chain above.

```bash
kubectl get secret team-a-network-conn -n default -o jsonpath='{.data}' | jq 'keys'
```

Still just `["subnetId","vpcId"]` — the last four steps never declared `connectionDetails`, which is expected: only resources whose identifiers the app team actually needs (VPC, Subnet) surface into the secret.

**Cross-verify against AWS:**

```bash
aws ec2 describe-vpcs --filters "Name=cidr,Values=10.0.0.0/16"
```

---

## Module 8: Cleanup

Deleting the claim cascades through the XR to every composed Managed Resource, regardless of which stage introduced it:

```bash
kubectl delete -f claim-vpcnetwork.yaml
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

## Quick Reference: What Changed at Each Stage

| Stage | MR added | New claim parameter(s) | New connection key | Pipeline step count |
|---|---|---|---|---|
| 1 | VPC | `region`, `vpcCidrBlock` | `vpcId` | 1 |
| 2 | Subnet | `subnetCidrBlock`, `availabilityZone` | `subnetId` | 2 |
| 3 | InternetGateway | — | — | 3 |
| 4 | RouteTable | — | — | 4 |
| 5 | RouteTableAssociation | — | — | 5 |
| 6 | SecurityGroup | — | — | 6 |

| Purpose | Command pattern |
|---|---|
| Confirm a step was appended, not replacing prior steps | `kubectl describe composition <name>` — count `step:` entries |
| Confirm the XR picked up a newly appended resource | `kubectl describe <Xkind> <name>` — count `Resource Refs` |
| Confirm which keys a connection secret currently has | `kubectl get secret <name> -o jsonpath='{.data}' \| jq 'keys'` |
| Force reconciliation after editing only the Composition | `kubectl apply -f <claim-file>` (re-apply, even unchanged) |
