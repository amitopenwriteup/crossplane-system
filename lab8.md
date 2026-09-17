# Lab: Launch an EC2 Instance with Crossplane Managed Resources Only 

This lab skips composition entirely. You apply raw Managed Resources (MRs) directly — `KeyPair`, `SecurityGroup`, `Instance` — the same style as the very first lab's `VPC`/`Subnet`, just aimed at launching a VM into the AWS account's **default** VPC and subnet in `us-east-2`, using an Ubuntu AMI.

> **Prerequisite:** `provider-aws-ec2` installed and healthy (from the base lab). AWS CLI configured locally with credentials that can read EC2/SSM in `us-east-2` — you'll use it once, outside Crossplane, to look up IDs.

---

## Module 1: Get the Default VPC and Subnet Through Crossplane; the AMI You Still Look Up Yourself

Two different answers here, worth separating clearly:

**Default VPC / default subnet — yes, Crossplane can get these for you, no AWS CLI needed.** `provider-aws-ec2` ships `DefaultVPC` and `DefaultSubnet` Managed Resource kinds — the same idea as Terraform's `aws_default_vpc` / `aws_default_subnet`. You don't give them an ID; you give a region (and an AZ, for the subnet), and Crossplane **adopts** the account's actual default VPC/subnet — it does not create a new one — then reads its real attributes back into `status.atProvider`. This is a genuine "get details of the default VPC via Crossplane."

```bash
kubectl explain defaultvpc.spec.forProvider
kubectl explain defaultsubnet.spec.forProvider
```

```bash
vi defaultvpc-team-a.yaml
```

```yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: DefaultVPC
metadata:
  name: team-a-default-vpc
spec:
  forProvider:
    region: us-east-2
  providerConfigRef:
    name: default
```

```bash
vi defaultsubnet-team-a.yaml
```

```yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: DefaultSubnet
metadata:
  name: team-a-default-subnet
spec:
  forProvider:
    region: us-east-2
    availabilityZone: us-east-2a
  providerConfigRef:
    name: default
```

```bash
kubectl apply -f defaultvpc-team-a.yaml -f defaultsubnet-team-a.yaml
kubectl get defaultvpc,defaultsubnet
```

Wait for `READY`/`SYNCED: True`, then read the real IDs straight out of `status`:

```bash
export DEFAULT_VPC_ID=$(kubectl get defaultvpc team-a-default-vpc -o jsonpath='{.status.atProvider.id}')
export DEFAULT_SUBNET_ID=$(kubectl get defaultsubnet team-a-default-subnet -o jsonpath='{.status.atProvider.id}')
echo "VPC:    $DEFAULT_VPC_ID"
echo "Subnet: $DEFAULT_SUBNET_ID"
```

`kubectl describe defaultvpc team-a-default-vpc` also shows the real CIDR block, default network ACL ID, and default route table ID — all read, not created, since AWS already provisions exactly one default VPC and one default subnet per AZ per account.

> Deleting a `DefaultVPC`/`DefaultSubnet` MR does **not** delete the actual default VPC/subnet in AWS unless you explicitly opt into `forceDestroy` — by design, these Kinds exist to *observe and lightly manage* something AWS creates for you automatically, not to own its lifecycle the way a plain `VPC`/`Subnet` MR does.

**Ubuntu AMI — no, there's still no MR for this.** AMI discovery by filter (owner + name pattern + "most recent") is a Terraform *data source* (`data "aws_ami"`), and Upbound's generator only produces Managed Resources from Terraform *resources* — things with a create/delete lifecycle. An `AMI` MR does exist, but only for building or copying a custom AMI you own; it has no "find the latest Canonical-published Ubuntu image" mode. So this one still comes from outside Crossplane — the AWS-native equivalent of a data-source lookup is Canonical's public SSM parameter:

```bash
export UBUNTU_AMI_ID=$(aws ssm get-parameter \
  --region us-east-2 \
  --name /aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id \
  --query "Parameter.Value" --output text)
echo "AMI: $UBUNTU_AMI_ID"
```

(If you'd rather see this pulled in *without* leaving Crossplane entirely, the only path is a custom Composition **Function** that calls the AWS SDK's `DescribeImages` at pipeline time and patches the result into the XR — which needs XR/Composition, so it's out of scope for this MR-only lab.)

---

## Module 2: Generate an SSH Key Pair

Generate a fresh key pair locally — never reuse a personal key for a throwaway lab VM.

```bash
ssh-keygen -t ed25519 -f ./team-a-key -C "team-a@platform" -N ""
```

- `-t ed25519` — modern, short key type (use `-t rsa -b 4096` instead if your environment needs RSA compatibility)
- `-f ./team-a-key` — writes `./team-a-key` (private) and `./team-a-key.pub` (public) into the current directory
- `-C "team-a@platform"` — just a label/comment embedded in the public key, cosmetic only
- `-N ""` — empty passphrase, so the lab's `ssh` step doesn't prompt (fine for a disposable lab VM; don't do this for anything long-lived)

```bash
chmod 400 ./team-a-key
cat ./team-a-key.pub
```

Copy that `ssh-ed25519 AAAA... team-a@platform` output — it goes directly into the `KeyPair` MR next.

---

## Module 3: `KeyPair` MR — Import the Public Key into AWS

This doesn't generate a key in AWS — it registers the *public* key you already generated, exactly like `aws ec2 import-key-pair` would.

```bash
kubectl explain keypair.spec.forProvider
```

```bash
vi keypair-team-a.yaml
```

```yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: KeyPair
metadata:
  name: team-a-keypair
spec:
  forProvider:
    region: us-east-2
    publicKey: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... team-a@platform"   # ← paste your ./team-a-key.pub contents here
  providerConfigRef:
    name: default
```

```bash
kubectl apply -f keypair-team-a.yaml
kubectl get keypair team-a-keypair
```

Confirm `READY` and `SYNCED` are `True`.

```bash
aws ec2 describe-key-pairs --region us-east-2 --key-names team-a-keypair
```

---

## Module 4: `SecurityGroup` + `SecurityGroupRule` MRs — Open Port 22 in the Default VPC

In this provider version, `SecurityGroup` only creates the empty group — no inline `ingress`/`egress` block. Rules are their own `SecurityGroupRule` Kind, one object per rule, the same way AWS's own API separates `CreateSecurityGroup` from `AuthorizeSecurityGroupIngress`/`Egress`. (If you're on an older provider build that *does* support inline `ingress`/`egress`, `kubectl explain securitygroup.spec.forProvider` will show it — check there first rather than assuming either shape.)

```bash
kubectl explain securitygroup.spec.forProvider
kubectl explain securitygrouprule.spec.forProvider
```

**The bare security group, pointed at the default VPC's real ID:**

```bash
vi securitygroup-ssh.yaml
```

```yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: SecurityGroup
metadata:
  name: team-a-ssh-sg
  labels:
    purpose: ssh-access
spec:
  forProvider:
    region: us-east-2
    vpcId: vpc-REPLACE_WITH_DEFAULT_VPC_ID     # ← $DEFAULT_VPC_ID from Module 1
    description: Allow inbound SSH for the lab VM
  providerConfigRef:
    name: default
```

The `labels: purpose: ssh-access` is deliberate: without an owning XR, `matchControllerRef` isn't available on the Instance MR below, so the Instance will find this security group with a **label selector** instead.

**The ingress rule (port 22), referencing the group above by name, not by hardcoded ID:**

```bash
vi securitygrouprule-ssh-ingress.yaml
```

```yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: SecurityGroupRule
metadata:
  name: team-a-ssh-ingress
spec:
  forProvider:
    region: us-east-2
    type: ingress
    fromPort: 22
    toPort: 22
    protocol: tcp
    cidrBlocks:
      - 0.0.0.0/0
    securityGroupIdRef:
      name: team-a-ssh-sg
  providerConfigRef:
    name: default
```

**The egress rule (allow all outbound), same pattern:**

```bash
vi securitygrouprule-ssh-egress.yaml
```

```yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: SecurityGroupRule
metadata:
  name: team-a-ssh-egress
spec:
  forProvider:
    region: us-east-2
    type: egress
    fromPort: 0
    toPort: 0
    protocol: "-1"
    cidrBlocks:
      - 0.0.0.0/0
    securityGroupIdRef:
      name: team-a-ssh-sg
  providerConfigRef:
    name: default
```

> `0.0.0.0/0` on port 22 is "lab only" — lock this to your own IP (`<your-ip>/32`) for anything real.

```bash
kubectl apply -f securitygroup-ssh.yaml
kubectl apply -f securitygrouprule-ssh-ingress.yaml -f securitygrouprule-ssh-egress.yaml
kubectl get securitygroup,securitygrouprule
```

Confirm `READY`/`SYNCED: True` on all three before moving on — a rule can't attach until the group it references exists and is synced.

```bash
aws ec2 describe-security-groups --region us-east-2 --group-names team-a-ssh-sg \
  --query "SecurityGroups[0].IpPermissions"
```

---

## Module 5: `Instance` MR — Launch the VM

```bash
kubectl explain instance.spec.forProvider
```

```bash
vi instance-team-a.yaml
```

```yaml
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Instance
metadata:
  name: team-a-vm
spec:
  forProvider:
    region: us-east-2
    ami: ami-REPLACE_WITH_UBUNTU_AMI_ID          # ← $UBUNTU_AMI_ID from Module 1
    instanceType: t3.micro
    subnetId: subnet-REPLACE_WITH_DEFAULT_SUBNET_ID   # ← $DEFAULT_SUBNET_ID from Module 1
    associatePublicIpAddress: true
    vpcSecurityGroupIdSelector:
      matchLabels:
        purpose: ssh-access
    keyName: team-a-keypair              # ← plain string, not a Ref/Selector
    tags:
      Name: team-a-vm
  providerConfigRef:
    name: default
```

> **Before applying, confirm the schema on your cluster.** Not every field on an upjet-generated MR gets a matching `*Ref`/`*Selector` pair — it depends on how the provider was generated, and it varies by version. Run `kubectl explain instance.spec.forProvider --recursive | grep -i key` first: if it shows only a plain `keyName` string (no `keyNameRef`/`keyNameSelector`), use `keyName` as above, set to the value your `KeyPair` MR's AWS key actually has — which, if you never set a `crossplane.io/external-name` annotation on it, is just its Kubernetes object name (`team-a-keypair`). Confirm with:
>
> ```bash
> kubectl get keypair team-a-keypair -o jsonpath='{.metadata.annotations.crossplane\.io/external-name}'
> ```

One cross-resource reference still applies here, since it's a genuine list-selector field on this CRD:

- **`vpcSecurityGroupIdSelector.matchLabels`** — finds any `SecurityGroup` MR carrying the label `purpose: ssh-access` and resolves its real AWS security group ID into `vpcSecurityGroupIds` at apply time. This works standalone, without any Composition, XR, or XRD involved.

```bash
kubectl apply -f instance-team-a.yaml
kubectl get instance team-a-vm
```

Wait for `READY: True`, `SYNCED: True` — this can take a minute or two for a fresh EC2 launch.

---

## Module 6: Verify

```bash
kubectl describe instance team-a-vm
```

Look at `status.atProvider` for `publicIp` / `publicDns`.

```bash
export VM_IP=$(kubectl get instance team-a-vm -o jsonpath='{.status.atProvider.publicIp}')
echo "$VM_IP"
```

**Cross-verify against AWS directly:**

```bash
aws ec2 describe-instances --region us-east-2 \
  --filters "Name=tag:Name,Values=team-a-vm" \
  --query "Reservations[0].Instances[0].[InstanceId,State.Name,PublicIpAddress]" \
  --output table
```

---

## Module 7: Cleanup

```bash
kubectl delete -f instance-team-a.yaml
kubectl delete -f securitygrouprule-ssh-ingress.yaml -f securitygrouprule-ssh-egress.yaml
kubectl delete -f securitygroup-ssh.yaml
kubectl delete -f keypair-team-a.yaml
```

```bash
kubectl get instance,securitygroup,securitygrouprule,keypair
```

Confirm all four are gone.

```bash
kubectl delete -f defaultvpc-team-a.yaml -f defaultsubnet-team-a.yaml
```

This only removes the Kubernetes-side `DefaultVPC`/`DefaultSubnet` objects — it does **not** delete AWS's actual default VPC/subnet, since Crossplane adopted rather than created them (no `forceDestroy` was set). The account's default network stays exactly as it was before this lab.



roup); a plain literal value matching the target MR's external name where no `*Ref`/`*Selector` exists (KeyPair's `keyName`) — `matchControllerRef` is composition-only |
| Get the launched VM's public IP | `kubectl get instance <name> -o jsonpath='{.status.atProvider.publicIp}'` |
