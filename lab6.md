# Lab: Crossplane XR & XRD — Wrapping the VPC Stack into a Single API

The previous lab created six separate Managed Resources by hand: `VPC`, `Subnet`, `InternetGateway`, `RouteTable`, `RouteTableAssociation`, and `SecurityGroup`. A platform team doesn't want application teams applying six YAML files and wiring `*Ref` fields together every time they need a network. This lab wraps all six into **one Composite Resource Definition (XRD)** and **one Composition**, so a consumer applies a single `VPCNetwork` claim and gets the whole stack.

> **Prerequisite:** Complete Module 1 and 2 of the previous lab first (`provider-aws-ec2` installed and healthy). This lab doesn't touch the underlying Managed Resources — it adds an abstraction layer on top of them.

---

## Module 1: Concepts — XRD, Composition, XR, and Claim

Four objects work together:

| Object | What it is | Who creates it |
|---|---|---|
| **XRD** (`CompositeResourceDefinition`) | Defines the schema of your new API (e.g. `VPCNetwork`, with fields like `region` and `cidrBlock`) | Platform team, once |
| **Composition** | Maps that schema onto real Managed Resources (VPC, Subnet, etc.) and patches values between them | Platform team, once (can have several per XRD) |
| **XR** (Composite Resource) | The cluster-scoped instance Crossplane creates when a claim is submitted — the "assembled" object | Crossplane, automatically |
| **Claim** | The namespaced object an application team actually applies — this is the "single API" they interact with | Application team |

```
Claim (namespaced, what app teams touch)
   │  1:1
   ▼
XR / Composite Resource (cluster-scoped, Crossplane-managed)
   │  composed of
   ▼
VPC + Subnet + InternetGateway + RouteTable + RouteTableAssociation + SecurityGroup
   (the Managed Resources from the previous lab)
```

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

## Module 3: Author the Composition — Map the API to Real Resources

**Explain first:**

```bash
kubectl explain composition.spec
kubectl explain composition.spec.resources
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
        - fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.region
        - fromFieldPath: spec.parameters.vpcCidrBlock
          toFieldPath: spec.forProvider.cidrBlock
      connectionDetails:
        - name: vpcId
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
        - fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.region
        - fromFieldPath: spec.parameters.subnetCidrBlock
          toFieldPath: spec.forProvider.cidrBlock
        - fromFieldPath: spec.parameters.availabilityZone
          toFieldPath: spec.forProvider.availabilityZone
      connectionDetails:
        - name: subnetId
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
        - fromFieldPath: spec.parameters.region
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
        - fromFieldPath: spec.parameters.region
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
        - fromFieldPath: spec.parameters.region
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
        - fromFieldPath: spec.parameters.region
          toFieldPath: spec.forProvider.region
```

Two things worth calling out versus the hand-written manifests from the last lab:

- **`matchControllerRef: true` replaces every hardcoded `vpcIdRef` / `gatewayIdRef` / `subnetIdRef`.** Since all six resources here are owned by the same XR, Crossplane can automatically wire a Subnet to *its* VPC without you naming it — this is what makes the Composition reusable across many `VPCNetwork` claims instead of just one hardcoded stack.
- **`connectionDetails`** surfaces `status.atProvider.id` from the VPC and Subnet up into a Kubernetes Secret, which is how the `vpcId` / `subnetId` keys declared in the XRD's `connectionSecretKeys` actually get populated.

```bash
kubectl apply -f composition-vpcnetwork.yaml
kubectl get composition
```

**Confirm it's valid against the XRD:**

```bash
kubectl describe composition vpcnetwork-aws
```

Look for `Synced: True` — a mismatched `compositeTypeRef` or unknown field path here is the most common error at this step.

```bash
kubectl api-resources | grep composition
```

---

## Module 4: Consume the API — Submit a Claim

This is the payload an application team writes. Notice it never mentions `VPC`, `Subnet`, or any provider-specific field — only the parameters your XRD schema exposed.

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

**Watch the pod(s) — still no new provider pod, same as the last lab:**

```bash
kubectl get pods -n crossplane-system
```

One `provider-aws-ec2` pod continues to reconcile every resource this claim creates — the Composition changes *what gets created*, not *what reconciles it*.

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

Confirm all six are gone, then remove the platform-level objects:

```bash
kubectl delete -f composition-vpcnetwork.yaml
kubectl delete -f xrd-vpcnetwork.yaml
kubectl api-resources | grep platform.example.org
```

The last command should return nothing once the XRD is gone.

---

## Quick Reference: New Objects Introduced in This Lab

| Purpose | Command pattern |
|---|---|
| Define a new composite API | `kubectl explain compositeresourcedefinition.spec` |
| See the new claim-facing Kind's schema | `kubectl explain <claimkind>.spec.parameters` |
| Map schema fields to Managed Resources | `kubectl explain composition.spec.resources` |
| List claims / composites | `kubectl get <claimkind>,<X composite kind>` |
| Trace claim → XR → Managed Resources | `kubectl describe <claimkind> <name>` then `kubectl describe <Xkind> <name>` |
| Read connection secret values | `kubectl get secret <name> -o jsonpath='{.data.<key>}' \| base64 -d` |
