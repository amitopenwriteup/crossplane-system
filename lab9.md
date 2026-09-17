# Lab: Crossplane XR & XRD — Wrapping the EC2 Instance Stack into a Single API

The previous lab created several separate Managed Resources by hand: `DefaultVPC`, `DefaultSubnet`, `KeyPair`, `SecurityGroup`, two `SecurityGroupRule`s, and `Instance`. A platform team doesn't want application teams applying six-plus YAML files and wiring `*Selector`/external-name values together every time they need a VM. This lab wraps all of it into **one Composite Resource Definition (XRD)** and **one Composition**, so a consumer applies a single `EC2Instance` claim and gets a running VM.

> **Prerequisite:** Complete the MR-only EC2 lab first (`provider-aws-ec2` installed and healthy, and you've confirmed the real field names on `SecurityGroup`, `SecurityGroupRule`, and `Instance` for your provider version — this lab reuses those exact fields). If you've already done the VPC `XRD`/`Composition` lab, `function-patch-and-transform` is already installed — skip Module 3a, it's idempotent to re-apply anyway.
>
> **Important field-support caveat, carried over from the MR-only lab:** `Instance.spec.forProvider.keyName` has no `keyNameRef`/`keyNameSelector` counterpart on this provider version — it's a plain string. That means the usual "wire two composed resources together" trick (a `*Selector` with `matchControllerRef: true`) **doesn't work** for the key pair. Module 3 below uses a different technique to work around exactly this: patching the same literal value (the XR's own generated name) into both the `KeyPair`'s external-name annotation and the `Instance`'s `keyName` field, so they end up equal without either one referencing the other.

---

## Module 1: Concepts — XRD, Composition, XR, and Claim

Same four objects as the VPC lab, plus the same Function dependency:



| Value | Available via a Crossplane MR? | What the claim must supply instead |
|---|---|---|
| Default VPC's real ID | **Yes** — `DefaultVPC` MR | Nothing — composed automatically |
| Default subnet's real ID | **Yes** — `DefaultSubnet` MR | Just `region` + `availabilityZone` |
| Ubuntu/any AMI ID | **No** — no lookup-by-filter MR exists | `amiId` — the caller must resolve this themselves (AWS CLI/SSM) and pass it in |
| SSH public key material | **No** — Crossplane doesn't run `ssh-keygen` | `sshPublicKey` — the caller generates a key pair locally and pastes the public half in |

---

## Module 2: Author the XRD — Define the `EC2Instance` API

**Explain the schema you're about to write against:**

```bash
kubectl explain compositeresourcedefinition.spec
kubectl explain compositeresourcedefinition.spec.versions
```

```bash
vi xrd-ec2instance.yaml
```

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xec2instances.aws.platform.example.org
spec:
  group: aws.platform.example.org
  names:
    kind: XEC2Instance
    plural: xec2instances
  claimNames:
    kind: EC2Instance
    plural: ec2instances
  connectionSecretKeys:
    - instanceId
    - publicIp
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
                    availabilityZone:
                      type: string
                      description: AZ whose default subnet the instance launches into
                    amiId:
                      type: string
                      description: AMI ID to launch — Crossplane has no AMI lookup MR, so the caller must resolve this themselves
                    instanceType:
                      type: string
                      description: EC2 instance type
                      default: t3.micro
                    sshPublicKey:
                      type: string
                      description: Public half of an SSH key pair, e.g. the contents of an id_ed25519.pub file
                    allowedSshCidr:
                      type: string
                      description: CIDR range allowed to SSH in on port 22
                      default: 0.0.0.0/0
                  required:
                    - region
                    - availabilityZone
                    - amiId
                    - sshPublicKey
              required:
                - parameters
            status:
              type: object
              properties:
                instanceId:
                  type: string
                publicIp:
                  type: string
```

> `instanceType` and `allowedSshCidr` have schema-level `default`s, so the claim can omit them entirely for the common case — only `region`, `availabilityZone`, `amiId`, and `sshPublicKey` are actually required.

```bash
kubectl apply -f xrd-ec2instance.yaml
kubectl get xrd
```

**Confirm it established and offered the claim:**

```bash
kubectl describe xrd xec2instances.aws.platform.example.org
```

Look for `Established: True` and `Offered: True`.

**Confirm the new API landed:**

```bash
kubectl api-resources | grep platform.example.org
kubectl explain ec2instance.spec.parameters
```

---

## Module 3: Install the Function, then Author the Composition

### 3a. Install `function-patch-and-transform` (skip if already installed)

```bash
kubectl get functions.pkg.crossplane.io
```

If it's not already there:

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
kubectl get pods -n crossplane-system | grep function-patch-and-transform
```

Wait for `INSTALLED: True` and `HEALTHY: True`.

### 3b. Author the Composition

```bash
kubectl explain composition.spec.pipeline
```

```bash
vi composition-ec2instance.yaml
```

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: ec2instance-aws
  labels:
    provider: aws
spec:
  compositeTypeRef:
    apiVersion: aws.platform.example.org/v1alpha1
    kind: XEC2Instance
  mode: Pipeline
  pipeline:
    - step: patch-and-transform
      functionRef:
        name: function-patch-and-transform
      input:
        apiVersion: pt.fn.crossplane.io/v1beta1
        kind: Resources
        resources:
          - name: default-vpc
            base:
              apiVersion: ec2.aws.upbound.io/v1beta1
              kind: DefaultVPC
              spec:
                forProvider: {}
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.region

          - name: default-subnet
            base:
              apiVersion: ec2.aws.upbound.io/v1beta1
              kind: DefaultSubnet
              spec:
                forProvider: {}
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.region
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.availabilityZone
                toFieldPath: spec.forProvider.availabilityZone

          - name: keypair
            base:
              apiVersion: ec2.aws.upbound.io/v1beta1
              kind: KeyPair
              spec:
                forProvider: {}
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.region
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.sshPublicKey
                toFieldPath: spec.forProvider.publicKey
              # ↓ the keyName workaround: force this MR's AWS-side name to equal
              # the XR's own (auto-generated) Kubernetes name
              - type: FromCompositeFieldPath
                fromFieldPath: metadata.name
                toFieldPath: metadata.annotations[crossplane.io/external-name]

          - name: security-group
            base:
              apiVersion: ec2.aws.upbound.io/v1beta1
              kind: SecurityGroup
              spec:
                forProvider:
                  description: Security group managed via EC2Instance composition
                  vpcIdSelector:
                    matchControllerRef: true
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.region

          - name: security-group-rule-ingress
            base:
              apiVersion: ec2.aws.upbound.io/v1beta1
              kind: SecurityGroupRule
              spec:
                forProvider:
                  type: ingress
                  fromPort: 22
                  toPort: 22
                  protocol: tcp
                  cidrBlocks:
                    - 0.0.0.0/0
                  securityGroupIdSelector:
                    matchControllerRef: true
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.region
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.allowedSshCidr
                toFieldPath: spec.forProvider.cidrBlocks[0]

          - name: security-group-rule-egress
            base:
              apiVersion: ec2.aws.upbound.io/v1beta1
              kind: SecurityGroupRule
              spec:
                forProvider:
                  type: egress
                  fromPort: 0
                  toPort: 0
                  protocol: "-1"
                  cidrBlocks:
                    - 0.0.0.0/0
                  securityGroupIdSelector:
                    matchControllerRef: true
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.region

          - name: instance
            base:
              apiVersion: ec2.aws.upbound.io/v1beta1
              kind: Instance
              spec:
                forProvider:
                  associatePublicIpAddress: true
                  subnetIdSelector:
                    matchControllerRef: true
                  vpcSecurityGroupIdSelector:
                    matchControllerRef: true
            patches:
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.region
                toFieldPath: spec.forProvider.region
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.amiId
                toFieldPath: spec.forProvider.ami
              - type: FromCompositeFieldPath
                fromFieldPath: spec.parameters.instanceType
                toFieldPath: spec.forProvider.instanceType
              # ↓ same literal value as the KeyPair's external-name above
              - type: FromCompositeFieldPath
                fromFieldPath: metadata.name
                toFieldPath: spec.forProvider.keyName
              - type: FromCompositeFieldPath
                fromFieldPath: metadata.name
                toFieldPath: spec.forProvider.tags[Name]
            connectionDetails:
              - name: instanceId
                type: FromFieldPath
                fromFieldPath: status.atProvider.id
              - name: publicIp
                type: FromFieldPath
                fromFieldPath: status.atProvider.publicIp
```

Five things worth calling out versus the hand-written MRs from the last lab:

- **`default-vpc` / `default-subnet` use `DefaultVPC`/`DefaultSubnet`, not `VPC`/`Subnet`** — same adopt-don't-create behavior as when you ran them standalone; nothing about wrapping them in a Composition changes that.
- **`matchControllerRef: true` replaces every hardcoded ID** for `vpcIdSelector`, `subnetIdSelector`, and `vpcSecurityGroupIdSelector` — since every composed resource here is owned by the same XR, Crossplane wires them together automatically. This assumes those `*Selector` fields exist on your provider version — the same caveat from the MR-only lab applies: run `kubectl explain <kind>.spec.forProvider --recursive | grep -i selector` against your installed CRDs before trusting this list.
- **The `keyName` workaround is the one asymmetric piece.** Because `Instance.spec.forProvider.keyName` has no `Ref`/`Selector` variant on this provider version, it can't be wired via `matchControllerRef` the way everything else is. Instead, both the `KeyPair`'s `crossplane.io/external-name` annotation and the `Instance`'s `keyName` are patched from the *same* source field — the XR's own `metadata.name` — so they land on the same literal string independently, without one resource needing to reference the other.
- **`cidrBlocks[0]`** is an array-index patch — `allowedSshCidr` is a single string on the claim, but `SecurityGroupRule.spec.forProvider.cidrBlocks` is a list, so the patch targets its first element directly rather than replacing the whole list.
- **`tags[Name]`** is a map-key patch, same mechanism as the array index above — it sets one key inside the `tags` object without needing the caller to supply a full tags map.

```bash
kubectl apply -f composition-ec2instance.yaml
kubectl describe composition ec2instance-aws
```

Look for `Synced: True`.

---

## Module 4: Consume the API — Submit a Claim

Generate a throwaway SSH key pair first, same as the MR-only lab:

```bash
ssh-keygen -t ed25519 -f ./team-a-key -C "team-a@platform" -N ""
cat ./team-a-key.pub
```

Look up an AMI ID the same way as before — this step still can't happen inside Crossplane:

```bash
export UBUNTU_AMI_ID=$(aws ssm get-parameter \
  --region us-east-2 \
  --name /aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id \
  --query "Parameter.Value" --output text)
echo "$UBUNTU_AMI_ID"
```

```bash
kubectl explain ec2instance.spec.parameters
```

```bash
vi claim-ec2instance.yaml
```

```yaml
apiVersion: aws.platform.example.org/v1alpha1
kind: EC2Instance
metadata:
  name: team-a-vm
  namespace: default
spec:
  parameters:
    region: us-east-2
    availabilityZone: us-east-2a
    amiId: ami-05b2e9c1e4d422698          # ← $UBUNTU_AMI_ID
    instanceType: t3.micro
    sshPublicKey: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... team-a@platform"   # ← ./team-a-key.pub contents
    allowedSshCidr: 0.0.0.0/0
  compositionRef:
    name: ec2instance-aws
  writeConnectionSecretToRef:
    name: team-a-vm-conn
```

Notice this claim never mentions `DefaultVPC`, `DefaultSubnet`, `KeyPair`, `SecurityGroup`, `SecurityGroupRule`, or `Instance` — only the parameters the XRD schema exposed.

```bash
kubectl apply -f claim-ec2instance.yaml
kubectl get ec2instance -n default
```

```bash
kubectl get pods -n crossplane-system
```

Same pods as before — no new one for this claim, regardless of how many `EC2Instance` claims you submit.

---

## Module 5: Verify the Full Chain

```bash
kubectl get ec2instance,xec2instance,defaultvpc,defaultsubnet,keypair,securitygroup,securitygrouprule,instance
```

Confirm `READY` and `SYNCED` are `True` across every row — this can take a minute or two for the `Instance` while EC2 launches.

**Trace claim → XR → Managed Resources:**

```bash
kubectl describe ec2instance team-a-vm -n default
kubectl get xec2instance -o wide
kubectl describe xec2instance <name-from-above>
```

If something's missing, check `status.conditions` for `FunctionPipelineFailed` first.

**Confirm the connection secret populated:**

```bash
kubectl get secret team-a-vm-conn -n default -o jsonpath='{.data.instanceId}' | base64 -d
kubectl get secret team-a-vm-conn -n default -o jsonpath='{.data.publicIp}' | base64 -d
```

**Confirm the `keyName` workaround actually landed correctly** — both values below should match:

```bash
kubectl get keypair -o jsonpath='{.items[0].metadata.annotations.crossplane\.io/external-name}'
kubectl get instance -o jsonpath='{.items[0].spec.forProvider.keyName}'
```

**Cross-verify against AWS:**

```bash
aws ec2 describe-instances --region us-east-2 \
  --filters "Name=tag:Name,Values=team-a-vm-*" \
  --query "Reservations[0].Instances[0].[InstanceId,State.Name,PublicIpAddress,KeyName]" \
  --output table
```

---

## Module 6: Cleanup

```bash
kubectl delete -f claim-ec2instance.yaml
kubectl get defaultvpc,defaultsubnet,keypair,securitygroup,securitygrouprule,instance
```

Confirm the `KeyPair`, `SecurityGroup`, `SecurityGroupRule`s, and `Instance` are gone. `DefaultVPC`/`DefaultSubnet` will also disappear from Kubernetes, but — same as before — the actual default VPC/subnet in AWS are untouched, since Crossplane only adopted them.

```bash
kubectl delete -f composition-ec2instance.yaml
kubectl delete -f xrd-ec2instance.yaml
kubectl api-resources | grep platform.example.org
```

Leave `function-patch-and-transform` installed if the VPC lab's Composition still depends on it.

```bash
rm -f ./team-a-key ./team-a-key.pub
```

---

## Quick Reference
