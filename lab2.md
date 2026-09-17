This step-by-step hands-on lab guides you through setting up the Crossplane AWS provider, AWS CLI on Linux, configuring credentials, and troubleshooting common issues.

---

1. **Prerequisites: Install AWS CLI on Linux:** Est. time: 3 mins.
Run the official AWS CLI v2 installation bundle on your Linux host:

```bash
# Download and extract the installer
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip

# Run installation
sudo ./aws/install

# Verify installation
aws --version

```

To verify success, ensure the output returns `aws-cli/2.x.x` without any missing dependency errors.


2. **Install AWS Provider in Crossplane:** Family vs Monolithic.
Crossplane provides two provider installation models:

* **Family Providers (Recommended):** Modern, modular approach. Installs a lightweight base provider plus service-specific sub-providers (e.g., `provider-aws-s3`, `provider-aws-ec2`) to conserve cluster resources and API limits.
* **Monolithic Provider (Legacy):** Installs all AWS CRDs in a single controller. Can cause cluster API slowness due to hundreds of CRDs.

Apply the family-based AWS provider manifest:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-s3
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v1.3.0
EOF

```

Verify the provider package status:

```bash
kubectl get provider.pkg.crossplane.io

```

Ensure `HEALTHY` reports `True` and `INSTALLED` reports `True`.


3. **Understand ProviderConfig Architecture:** Authentication Binding.
The **ProviderConfig** object serves as the operational bridge between managed resources and AWS authentication:

* It specifies **how** Crossplane authenticates (e.g., Static Access Keys, IRSA, Web Identity).
* It defines the default target AWS region.
* Individual custom resources (like an S3 Bucket) reference a specific `ProviderConfig` via `spec.providerConfigRef.name`.


4. **Configure AWS Credentials Secret:** Static Keys or IRSA.
Choose **Option A** for quick local/testing setups or **Option B** for production EKS environments.

**Option A: Static IAM User Access Keys (Testing/Local)**

1. Create a credential file format:

```bash
cat <<EOF > aws-credentials.ini
[default]
aws_access_key_id = YOUR_AWS_ACCESS_KEY_ID
aws_secret_access_key = YOUR_AWS_SECRET_ACCESS_KEY
EOF

```

2. Store the credential file in Kubernetes as a Secret:

```bash
kubectl create secret generic aws-secret \
  -n crossplane-system \
  --from-file=creds=./aws-credentials.ini

```

**Option B: IRSA / Web Identity (Production EKS)**

Annotate the Crossplane service account with your IAM Role ARN (no Kubernetes secrets required):

```bash
kubectl annotate serviceaccount provider-aws-s3-* \
  -n crossplane-system \
  eks.amazonaws.com/role-arn=arn:aws:iam::123456789012:role/crossplane-aws-role

```

Verify secret creation by running `kubectl get secret aws-secret -n crossplane-system`.


5. **Apply ProviderConfig & Verify Health:** Est. time: 2 mins.
Create the `ProviderConfig` object referencing the secret created in Step 4:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-secret
      key: creds
EOF

```

Verify health by submitting a basic test resource (an S3 bucket):

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: crossplane-healthcheck-bucket
spec:
  forProvider:
    region: us-east-1
  providerConfigRef:
    name: default
EOF

```

Check reconciliation status:

```bash
kubectl get bucket crossplane-healthcheck-bucket

```

Verify that both `READY` and `SYNCED` columns report `True`.


6. **Common Provider Troubleshooting:** Diagnostic Runbook.
**Issue 1: CRDs Not Appearing / Stuck Package**

* *Symptom:* `kubectl get bucket` returns "error: the server doesn't have a resource type".
* *Fix:* Check if package download or extraction failed due to cluster resource limits:

```bash
kubectl describe provider.pkg.crossplane.io provider-aws-s3

```

**Issue 2: Provider Pod Crash / OOMKilled**

* *Symptom:* Provider pod stays in `CrashLoopBackOff` or gets killed.
* *Fix:* Monolithic providers consume >2GB RAM. Check pod logs and increase deployment memory limits:

```bash
kubectl logs -n crossplane-system -l pkg.crossplane.io/provider=provider-aws-s3

```

**Issue 3: Authentication / Unreconciled Resources**

* *Symptom:* Resources stay `READY: False` with `AuthFailure` or `AccessDenied`.
* *Fix:* Verify event conditions on the resource:

```bash
kubectl describe bucket crossplane-healthcheck-bucket

```
