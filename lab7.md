# Lab: Crossplane XR & XRD — Wrapping the VPC Stack into a Single API

The previous lab created six separate Managed Resources by hand: `VPC`, `Subnet`, `InternetGateway`, `RouteTable`, `RouteTableAssociation`, and `SecurityGroup`. A platform team doesn't want application teams applying six YAML files and wiring `*Ref` fields together every time they need a network. This lab wraps all six into **one Composite Resource Definition (XRD)** and **one Composition**, so a consumer applies a single `VPCNetwork` claim and gets the whole stack.

> **Prerequisite:** Complete Module 1 and 2 of the previous lab first (`provider-aws-ec2` installed and healthy). This lab doesn't touch the underlying Managed Resources — it adds an abstraction layer on top of them.
>


---

## Module 1: Concepts — XRD, Composition, XR, and Claim

Four objects work together, plus one new supporting object this version of Crossplane requires:

| Object | What it is | Who creates it |
|---|---|---|
| **XRD** (`CompositeResourceDefinition`) | Defines the schema of your new API (e.g. `VPCNetwork`, with fields like `region` and `cidrBlock`) | Platform team, once |
| **Function** (`pkg.crossplane.io/v1beta1`) | An installed package (like a Provider) that a Composition's pipeline calls to actually build resources — e.g. `function-patch-and-transform` | Platform team, once, before writing Compositions |
| **Composition** | A `Pipeline` of function steps that maps the XRD schema onto real Managed Resources and patches values between them | Platform team, once (can have several per XRD) |
| **XR** (Composite Resource) | The cluster-scoped instance Crossplane creates when a claim is submitted — the "assembled" object | Crossplane, automatically |
| **Claim** | The namespaced object an application team actually applies — this is the "single API" they interact with | Application team |

> Think of a Function the same way you think of a Provider: a Provider gives Crossplane the CRDs + controller for *talking to a cloud API* (e.g. `provider-aws-ec2`); a Function gives Crossplane the logic for *building the desired resource set from a composite's spec* during a Composition's pipeline run. Both are installed as packages and both show up as pods in `crossplane-system`.

---

## Module 2: Author the XRD — Define the `VPCNetwork` API

**Explain the schema you're about to write against:**

```bash
kubectl explain compositeresourcedefinition.spec
kubectl explain compositeresourcedefinition.spec.versions
```

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
    - subnetId
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
                    subnetCidrBlock:
                      type: string
                      description: CIDR block for the public subnet
                    availabilityZone:
                      type: string
                      description: AZ for the subnet
                  required:
                    - region
                    - vpcCidrBlock
                    - subnetCidrBlock
                    - availabilityZone
              required:
                - parameters
            status:
              type: object
              properties:
                vpcId:
                  type: string
                subnetId:
                  type: string
```

> `names.kind` (`XVPCNetwork`) is the cluster-scoped XR Kind Crossplane creates internally. `claimNames.kind` (`VPCNetwork`) is the namespaced Kind your application teams will actually `kubectl apply`. The `X` prefix convention just marks "this is the composite, not the claim."

```bash
kubectl apply -f xrd-vpcnetwork.yaml
kubectl get xrd
```

**Confirm it established and offered the claim:**

```bash
kubectl describe xrd xvpcnetworks.aws.platform.example.org
```

Look for `Established: True` and `Offered: True` in the conditions, and — per the RBAC behavior you've already seen with providers — an `ApplyClusterRoles` event, since the XRD also triggers the RBAC manager to generate `crossplane-admin` / `-edit` / `-view` rules for the new `VPCNetwork` and `XVPCNetwork` kinds.

**Confirm the new API landed:**

```bash
kubectl api-resources | grep platform.example.org
```

You should see both `vpcnetworks` (namespaced) and `xvpcnetworks` (cluster-scoped) as new Kinds — this is the direct answer to "what API did I just create."

```bash
kubectl explain vpcnetwork.spec.parameters
```

This now reflects *your* schema, not an upstream provider's — confirming the XRD is the source of truth for this new Kind, the same way `provider-aws-ec2`'s CRDs were the source of truth for `vpc.spec.forProvider` in the last lab.

---

## Module 3: Install a Function, then Author the Composition

### 3a. Install `function-patch-and-transform`

Compositions in Pipeline mode don't build resources themselves — they hand the work to Functions. Install the one that replicates the classic base/patches/connectionDetails behavior:

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
kubectl get functions
```

**Confirm it's installed and healthy — same pattern as checking a Provider:**

```bash
kubectl get functions.pkg.crossplane.io
kubectl get pods -n crossplane-system | grep function-patch-and-transform
```

Wait for `INSTALLED: True` and `HEALTHY: True` before moving on — a Composition that references a Function which isn't ready yet will fail to compose with a "function not found" style error.

### 3b. Author the Composition

**Explain first:**

```bash
kubectl explain composition.spec
kubectl explain composition.spec.pipeline
```

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
    - step: patch-and-transform
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

Four things worth calling out versus the hand-written manifests from the last lab, and versus the classic-mode Composition this replaces:



```bash
kubectl apply -f composition-vpcnetwork.yaml
kubectl get composition
```

**Confirm it's valid against the XRD:**

```bash
kubectl describe composition vpcnetwork-aws
```

Look for `Synced: True` — a mismatched `compositeTypeRef`, an unknown field path, or a `functionRef` pointing at a Function that isn't installed/healthy are the most common errors at this step.

```bash
kubectl api-resources | grep composition
```

---

## Module 4: Consume the API — Submit a Claim

This is the payload an application team writes. Notice it never mentions `VPC`, `Subnet`, any provider-specific field, or the fact that a pipeline/function is doing the work underneath — only the parameters your XRD schema exposed.

**Explain first (this now reflects your schema exactly):**

```bash
kubectl explain vpcnetwork.spec.parameters
```

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
    subnetCidrBlock: 10.0.1.0/24
    availabilityZone: us-east-1a
  compositionRef:
    name: vpcnetwork-aws
  writeConnectionSecretToRef:
    name: team-a-network-conn
```

```bash
kubectl apply -f claim-vpcnetwork.yaml
kubectl get vpcnetwork -n default
```

**Watch the pod(s) — you'll now see the function pod join the picture:**

```bash
kubectl get pods -n crossplane-system
```

One `provider-aws-ec2` pod continues to reconcile every Managed Resource this claim creates, same as before. The `function-patch-and-transform` pod is called synchronously during each Composition run to decide *what* those resources should look like — it doesn't reconcile them itself and won't show sustained activity the way a Provider pod does.

---

## Module 5: Verify the Full Chain

**See the claim, the XR it created, and every Managed Resource it composed — in one pass:**

```bash
kubectl get vpcnetwork,xvpcnetwork,vpc,subnet,internetgateway,routetable,routetableassociation,securitygroup
```

Confirm `READY` and `SYNCED` are `True` across every row.

**Trace claim → XR → Managed Resources explicitly:**

```bash
kubectl describe vpcnetwork team-a-network -n default
```

Note the `Resource Refs` in the status pointing at the underlying `XVPCNetwork`, and that object's own `describe` pointing at all six Managed Resources it owns:

```bash
kubectl get xvpcnetwork -o wide
kubectl describe xvpcnetwork <name-from-above>
```

If something's missing here, check the XR's `status.conditions` for a `FunctionPipelineFailed` type before anything else — that's Pipeline mode's equivalent of a classic Composition patch error.

**Confirm the connection secret populated:**

```bash
kubectl get secret team-a-network-conn -n default -o jsonpath='{.data.vpcId}' | base64 -d
kubectl get secret team-a-network-conn -n default -o jsonpath='{.data.subnetId}' | base64 -d
```

**Cross-verify against AWS, same as before:**

```bash
aws ec2 describe-vpcs --filters "Name=tag:crossplane-kind,Values=vpc.ec2.aws.upbound.io"
```

Or match by CIDR:

```bash
aws ec2 describe-vpcs --filters "Name=cidr,Values=10.0.0.0/16"
```

---

## Module 6: Cleanup

Deleting the claim cascades through the XR to every composed Managed Resource — this is the main operational payoff of using a Composition:

```bash
kubectl delete -f claim-vpcnetwork.yaml
kubectl get vpc,subnet,internetgateway,routetable,routetableassociation,securitygroup
```

Confirm all six are gone, then remove the platform-level objects — Composition and XRD as before, plus the Function if no other Composition on the cluster still depends on it:

```bash
kubectl delete -f composition-vpcnetwork.yaml
kubectl delete -f xrd-vpcnetwork.yaml
kubectl delete -f function-patch-and-transform.yaml
kubectl api-resources | grep platform.example.org
```

The last command should return nothing once the XRD is gone.

---

