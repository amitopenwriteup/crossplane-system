# Lab: AWS VPC Provider Setup & Crossplane Platform Capabilities

This lab installs the Crossplane AWS VPC family provider, provisions a VPC with its supporting networking resources, cross-verifies everything against AWS, and then showcases Crossplane platform-level capabilities beyond the usual Managed Resources — **excluding Composition and CompositeResourceDefinition (XR/XRD)**, which are covered in a separate module.

Every section below now also shows:
- **`kubectl get pods`** — what pod(s) actually get created/changed in `crossplane-system` when a provider installs or is reconfigured
- **`kubectl api-resources`** — which API group/resource each provider enables on the cluster, so you can see the CRDs land
- **`kubectl explain`** — the live schema for each object/field, so you can see how the YAML was derived from the CRD
- **`kubectl describe`** — the full reconciled state (conditions, events, `status.atProvider`) after apply

> **Note:** Replace placeholder values (VPC name, CIDR block, region, availability zone) with your own in every manifest before applying.

> **Note:** Field names below reflect the `provider-aws-vpc` (part of `provider-family-aws`) v1.x CRD schema under the `ec2.aws.upbound.io` API group. Confirm exact fields for your installed version with `kubectl explain <resource>.spec.forProvider` before applying — this lab does that explicitly at every step.

---

## Module 1: Install & Verify the AWS VPC Provider

**Step 1 — Create the manifest with vi:**

```bash
vi provider-aws-vpc.yaml
```

Press `i`, type the following, then `Esc` and `:wq`:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-ec2
spec:
  package: xpkg.upbound.io/upbound/provider-aws-ec2:v1.3.0
```


**Step 2 — Apply it:**

```bash
kubectl apply -f provider-aws-vpc.yaml
```

**Step 3 — Watch the pod come up:**

```bash
kubectl get pods -n crossplane-system -w
```

You should see two pods appear over time:
- `provider-family-aws-<hash>` — the shared base pod (pulled in automatically as a dependency)
- `provider-aws-vpc-<hash>` — the VPC-specific provider pod

Ctrl+C once both show `Running` / `1/1 READY`.

**Step 4 — Verify provider health:**

```bash
kubectl get provider.pkg.crossplane.io
```

Confirm `HEALTHY` and `INSTALLED` are both `True`.

**Step 5 — Confirm which APIs the provider enabled:**

```bash
kubectl api-resources | grep ec2.aws.upbound.io
```

This is the direct answer to "what did installing this provider actually give me?" — every row is a new Kind (VPC, Subnet, InternetGateway, RouteTable, RouteTableAssociation, SecurityGroup, and many more) now served under the `ec2.aws.upbound.io` API group. Compare the count here to the (much smaller) list you'll actually use in Module 2 — this is the gap Module 4.1 (MRAP) later lets you shrink.

```bash
kubectl api-resources --api-group=ec2.aws.upbound.io -o wide
```

`-o wide` additionally shows the CRD's short names and whether it's namespaced.

---

## Module 2: Create Core VPC Networking Resources

### 2.1 Create the VPC

**Explain the schema first:**

```bash
kubectl explain vpc.spec.forProvider
kubectl explain vpc.spec.forProvider.tags
```

This is how the manifest below was derived — `region`, `cidrBlock`, `enableDnsSupport`, `enableDnsHostnames`, and `tags` are all properties `kubectl explain` will list directly.

```bash
vi vpc.yaml
```

```yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: VPC
metadata:
  name: crossplane-lab-vpc
spec:
  forProvider:
    region: us-east-1
    cidrBlock: 10.0.0.0/16
    enableDnsSupport: true
    enableDnsHostnames: true
    tags:
      Name: crossplane-lab-vpc
  providerConfigRef:
    name: default
```

```bash
kubectl apply -f vpc.yaml
kubectl get vpc crossplane-lab-vpc
```

**Confirm which API/Kind this came from:**

```bash
kubectl api-resources | grep -i vpc
```

**Inspect full reconciled state:**

```bash
kubectl describe vpc crossplane-lab-vpc
```

Check the `Status.Conditions` block for `Ready: True` / `Synced: True`, and `Status.At Provider` for the AWS-assigned `vpc-xxxx` ID.

### 2.2 Create a Public Subnet

**Explain first:**

```bash
kubectl explain subnet.spec.forProvider
kubectl explain subnet.spec.forProvider.vpcIdRef
```

```bash
vi subnet.yaml
```

```yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Subnet
metadata:
  name: crossplane-lab-subnet-public
spec:
  forProvider:
    region: us-east-1
    vpcIdSelector:
      matchLabels:
        crossplane.io/claim-name: crossplane-lab-vpc
    vpcIdRef:
      name: crossplane-lab-vpc
    cidrBlock: 10.0.1.0/24
    availabilityZone: us-east-1a
    mapPublicIpOnLaunch: true
    tags:
      Name: crossplane-lab-subnet-public
  providerConfigRef:
    name: default
```

> `vpcIdRef` lets this resource reference the VPC by its Crossplane object name instead of hardcoding the AWS-assigned VPC ID — Crossplane resolves it automatically once the VPC is `READY`.

```bash
kubectl apply -f subnet.yaml
kubectl get subnet crossplane-lab-subnet-public
```

**API check + full status:**

```bash
kubectl api-resources | grep -i subnet
kubectl describe subnet crossplane-lab-subnet-public
```

No new pod appears here — confirm that with `kubectl get pods -n crossplane-system`: one provider pod reconciles every Kind in its API group, so pod count doesn't scale with resource count.

### 2.3 Create an Internet Gateway

**Explain first:**

```bash
kubectl explain internetgateway.spec.forProvider
```

```bash
vi internetgateway.yaml
```

```yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: InternetGateway
metadata:
  name: crossplane-lab-igw
spec:
  forProvider:
    region: us-east-1
    vpcIdRef:
      name: crossplane-lab-vpc
    tags:
      Name: crossplane-lab-igw
  providerConfigRef:
    name: default
```

```bash
kubectl apply -f internetgateway.yaml
kubectl get internetgateway crossplane-lab-igw
```

```bash
kubectl api-resources | grep -i internetgateway
kubectl describe internetgateway crossplane-lab-igw
```

### 2.4 Create a Route Table with a Default Route

**Explain first (nested `route` array is the interesting part):**

```bash
kubectl explain routetable.spec.forProvider
kubectl explain routetable.spec.forProvider.route
```

```bash
vi routetable.yaml
```

```yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: RouteTable
metadata:
  name: crossplane-lab-rt
spec:
  forProvider:
    region: us-east-1
    vpcIdRef:
      name: crossplane-lab-vpc
    route:
      - destinationCidrBlock: 0.0.0.0/0
        gatewayIdRef:
          name: crossplane-lab-igw
    tags:
      Name: crossplane-lab-rt
  providerConfigRef:
    name: default
```

```bash
kubectl apply -f routetable.yaml
kubectl get routetable crossplane-lab-rt
```

```bash
kubectl api-resources | grep -i routetable
kubectl describe routetable crossplane-lab-rt
```

### 2.5 Associate the Route Table with the Subnet

**Explain first:**

```bash
kubectl explain routetableassociation.spec.forProvider
```

```bash
vi routetableassociation.yaml
```

```yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: RouteTableAssociation
metadata:
  name: crossplane-lab-rt-assoc
spec:
  forProvider:
    region: us-east-1
    subnetIdRef:
      name: crossplane-lab-subnet-public
    routeTableIdRef:
      name: crossplane-lab-rt
  providerConfigRef:
    name: default
```

```bash
kubectl apply -f routetableassociation.yaml
kubectl get routetableassociation crossplane-lab-rt-assoc
```

```bash
kubectl api-resources | grep -i routetableassociation
kubectl describe routetableassociation crossplane-lab-rt-assoc
```

### 2.6 Create a Security Group

**Explain first:**

```bash
kubectl explain securitygroup.spec.forProvider
```

```bash
vi securitygroup.yaml
```

```yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: SecurityGroup
metadata:
  name: crossplane-lab-sg
spec:
  forProvider:
    region: us-east-1
    vpcIdRef:
      name: crossplane-lab-vpc
    name: crossplane-lab-sg
    description: Lab security group managed by Crossplane
    tags:
      Name: crossplane-lab-sg
  providerConfigRef:
    name: default
```

```bash
kubectl apply -f securitygroup.yaml
kubectl get securitygroup crossplane-lab-sg
```

```bash
kubectl api-resources | grep -i securitygroup
kubectl describe securitygroup crossplane-lab-sg
```

**Sanity check across the whole module — still exactly one provider pod:**

```bash
kubectl get pods -n crossplane-system -l pkg.crossplane.io/provider=provider-aws-vpc
```

---

## Module 3: Cross-Verify Against AWS

**Via kubectl (Crossplane's view):**

```bash
kubectl get vpc,subnet,internetgateway,routetable,routetableassociation,securitygroup
```

Confirm `READY` and `SYNCED` are `True` across the board.

**Full status detail for every object in one pass:**

```bash
kubectl describe vpc,subnet,internetgateway,routetable,routetableassociation,securitygroup
```

**Via AWS CLI (the ground truth):**

```bash
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=crossplane-lab-vpc"
aws ec2 describe-subnets --filters "Name=tag:Name,Values=crossplane-lab-subnet-public"
aws ec2 describe-internet-gateways --filters "Name=tag:Name,Values=crossplane-lab-igw"
aws ec2 describe-route-tables --filters "Name=tag:Name,Values=crossplane-lab-rt"
aws ec2 describe-security-groups --filters "Name=tag:Name,Values=crossplane-lab-sg"
```

Match the AWS-assigned IDs (`vpc-xxxx`, `subnet-xxxx`, etc.) against what's shown in `kubectl describe <resource> <name>` under `status.atProvider`.

---

## Module 4: Beyond XR/XRD — Other Platform Capabilities

### 4.1 ManagedResourceActivationPolicy (MRAP) — Reduce the Provider's Footprint

A family provider like `provider-aws-vpc` ships CRDs for dozens of EC2/VPC-related resources, but a given cluster may only need a handful active. An MRAP activates only the MRs you list, leaving the rest inactive to reduce API server load.

**Baseline before applying — how many resources are currently active:**

```bash
kubectl api-resources | grep ec2.aws.upbound.io | wc -l
kubectl get managedresourcedefinitions | grep -c ec2.aws.upbound.io
```

**Explain the MRAP schema:**

```bash
kubectl explain managedresourceactivationpolicy.spec
kubectl explain managedresourceactivationpolicy.spec.activate
```

```bash
vi mrap-vpc-core.yaml
```

```yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: ManagedResourceActivationPolicy
metadata:
  name: vpc-core-only
spec:
  activate:
    - vpcs.ec2.aws.upbound.io
    - subnets.ec2.aws.upbound.io
    - internetgateways.ec2.aws.upbound.io
    - routetables.ec2.aws.upbound.io
    - routetableassociations.ec2.aws.upbound.io
    - securitygroups.ec2.aws.upbound.io
```

```bash
kubectl apply -f mrap-vpc-core.yaml
kubectl get managedresourceactivationpolicies
kubectl describe managedresourceactivationpolicy vpc-core-only
```

**Confirm activation state per CRD:**

```bash
kubectl get managedresourcedefinitions | grep ec2.aws.upbound.io
```

`ManagedResourceDefinition` (MRD) entries show the current activation `STATE` (`Active`/`Inactive`) per CRD — confirm only your listed six show `Active`.

**Confirm no pod churn — this is a control-plane-level gate, not a pod restart:**

```bash
kubectl get pods -n crossplane-system -l pkg.crossplane.io/provider=provider-aws-vpc
```

### 4.2 Usage — Protect the VPC from Accidental Deletion

A `Usage` records that one resource depends on another, and blocks deletion of the dependency until the dependent is gone — even though the Subnet already references the VPC via `vpcIdRef`, a `Usage` makes that protection explicit and enforced.

**Explain the schema:**

```bash
kubectl explain usage.spec
kubectl explain usage.spec.of
kubectl explain usage.spec.by
```

```bash
vi vpc-usage.yaml
```

```yaml
apiVersion: protection.crossplane.io/v1beta1
kind: Usage
metadata:
  name: subnet-uses-vpc
spec:
  of:
    apiVersion: ec2.aws.upbound.io/v1beta1
    kind: VPC
    resourceRef:
      name: crossplane-lab-vpc
  by:
    apiVersion: ec2.aws.upbound.io/v1beta1
    kind: Subnet
    resourceRef:
      name: crossplane-lab-subnet-public
```

```bash
kubectl apply -f vpc-usage.yaml
kubectl get usages
kubectl describe usage subnet-uses-vpc
```

**Confirm which API this comes from (a different group than the EC2 resources):**

```bash
kubectl api-resources | grep protection.crossplane.io
```

**Test it:** try deleting the VPC while the Subnet still exists —

```bash
kubectl delete vpc crossplane-lab-vpc
```

Expect this to hang/block (the VPC gets a deletion timestamp but won't finalize) until the Subnet — and the `Usage` — are removed first.

```bash
kubectl describe vpc crossplane-lab-vpc
```

Look for a `Usage`-related event/finalizer blocking deletion in the output.

### 4.3 DeploymentRuntimeConfig — Tune the Provider Pod

If the VPC provider pod needs more memory/CPU or more replicas (e.g. under heavy reconciliation load), a `DeploymentRuntimeConfig` customizes its Deployment without editing generated manifests directly.

**Baseline — current pod resources before the change:**

```bash
kubectl get pods -n crossplane-system -l pkg.crossplane.io/provider=provider-aws-vpc
kubectl describe pod -n crossplane-system -l pkg.crossplane.io/provider=provider-aws-vpc | grep -A4 Limits
```

**Explain the schema:**

```bash
kubectl explain deploymentruntimeconfig.spec
kubectl explain deploymentruntimeconfig.spec.deploymentTemplate
```

```bash
vi runtimeconfig-vpc.yaml
```

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  name: provider-aws-vpc-runtime
spec:
  deploymentTemplate:
    spec:
      replicas: 1
      template:
        spec:
          containers:
            - name: package-runtime
              resources:
                requests:
                  cpu: 100m
                  memory: 256Mi
                limits:
                  cpu: 500m
                  memory: 512Mi
```

Reference it from the Provider (edit `provider-aws-vpc.yaml` to add `spec.runtimeConfigRef`):

```yaml
spec:
  package: xpkg.upbound.io/upbound/provider-aws-vpc:v1.3.0
  runtimeConfigRef:
    name: provider-aws-vpc-runtime
```

```bash
kubectl apply -f runtimeconfig-vpc.yaml
kubectl apply -f provider-aws-vpc.yaml
kubectl get deploymentruntimeconfigs
kubectl api-resources | grep deploymentruntimeconfig
```

**Watch the pod get recreated with the new spec:**

```bash
kubectl get pods -n crossplane-system -l pkg.crossplane.io/provider=provider-aws-vpc -w
```

Ctrl+C once the new pod is `Running`.

```bash
kubectl describe pod -n crossplane-system -l pkg.crossplane.io/provider=provider-aws-vpc
```

Confirm the pod's resource requests/limits now match what you set.

### 4.4 ProviderRevision — Inspect Package Revision History

Every time a Provider's package version changes, Crossplane creates a new `ProviderRevision`. This is useful for auditing what's actually deployed and rolling back.

**Explain the schema:**

```bash
kubectl explain providerrevision.spec
kubectl explain providerrevision.status
```

```bash
kubectl api-resources | grep providerrevision
kubectl get providerrevisions
kubectl describe providerrevision <revision-name>
```

Look for the `DESIRED STATE` column (`Active` vs `Inactive`) — only one revision per provider should be `Active` at a time. Note in `describe` output that the revision that just changed (from adding `runtimeConfigRef` in 4.3) shows the new dependency/runtime association.

### 4.5 Lock — Inspect the Dependency Lock (Read-Only)

Crossplane maintains a single cluster-wide `Lock` object recording every installed package and its resolved dependencies (similar to a package-manager lockfile).

```bash
kubectl explain lock.packages
kubectl api-resources | grep -i lock
kubectl get lock lock -o yaml
```

Look for `provider-aws-vpc` and confirm its dependency on the shared `upbound-provider-family-aws` base package is listed.

### 4.6 EnvironmentConfig — Shared Values for Future Compositions

An `EnvironmentConfig` holds shared, non-secret values (default region, common tags, account IDs) that Compositions can later patch into multiple resources. You won't consume it here without a Composition, but creating one now shows the object exists and is queryable — useful groundwork if this lab's VPC resources get wrapped into a Composition later.

**Explain the schema:**

```bash
kubectl explain environmentconfig.data
```

```bash
vi envconfig-vpc.yaml
```

```yaml
apiVersion: apiextensions.crossplane.io/v1beta1
kind: EnvironmentConfig
metadata:
  name: vpc-lab-defaults
data:
  region: us-east-1
  defaultTags:
    Environment: lab
    ManagedBy: crossplane
```

```bash
kubectl apply -f envconfig-vpc.yaml
kubectl get environmentconfigs
kubectl describe environmentconfig vpc-lab-defaults
kubectl api-resources | grep environmentconfig
```

### 4.7 Operation — Run a One-Off Imperative Task

Unlike Managed Resources (which continuously reconcile), an `Operation` runs a Function once, for imperative tasks like a one-time cleanup or migration check. This is an alpha feature — treat it as a preview capability.

```bash
kubectl api-resources | grep -i operation
kubectl explain operation.spec
kubectl get operations
kubectl get cronoperations
kubectl get watchoperations
```

> These commands return empty in this lab since no Function or Operation has been defined — they're included to show the resource types exist and are queryable. Defining an actual Operation requires a Composition Function package, which is out of scope for this VPC-focused lab.

---

## Module 5: Cleanup

Delete in reverse dependency order — the `Usage` from 4.2 must go before the VPC can be deleted:

```bash
kubectl delete -f vpc-usage.yaml
kubectl delete -f envconfig-vpc.yaml
kubectl delete -f securitygroup.yaml
kubectl delete -f routetableassociation.yaml
kubectl delete -f routetable.yaml
kubectl delete -f internetgateway.yaml
kubectl delete -f subnet.yaml
kubectl delete -f vpc.yaml
kubectl delete -f runtimeconfig-vpc.yaml
kubectl delete -f mrap-vpc-core.yaml
```

**Confirm the provider pod is still healthy and no orphaned resources remain:**

```bash
kubectl get pods -n crossplane-system
kubectl get vpc,subnet,internetgateway,routetable,routetableassociation,securitygroup
```

---

## Quick Reference: vi Editor Basics

| Action | Command |
|---|---|
| Open/create a file | `vi filename` |
| Enter insert mode (to type/paste) | `i` |
| Exit insert mode | `Esc` |
| Save and quit | `:wq` then `Enter` |
| Quit without saving | `:q!` then `Enter` |

## Quick Reference: Verification Commands Used Throughout

| Purpose | Command pattern |
|---|---|
| See provider pods appear/change | `kubectl get pods -n crossplane-system` |
| Confirm HEALTHY/INSTALLED | `kubectl get provider.pkg.crossplane.io` |
| Find which API a resource belongs to | `kubectl api-resources \| grep <group-or-kind>` |
| See a CRD's full field schema | `kubectl explain <kind>.spec.forProvider` |
| See nested field schema | `kubectl explain <kind>.spec.forProvider.<field>` |
| Full reconciled object state | `kubectl describe <kind> <name>` |
| Ground-truth check in AWS | `aws ec2 describe-<resource> --filters ...` |
