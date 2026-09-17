# Crossplane AWS Provider — Hands-on Lab Guide

This step-by-step hands-on lab guides you through setting up the Crossplane AWS provider, AWS CLI on Linux, configuring credentials, and troubleshooting common issues.

YAML manifests are created with the **vi editor** and applied with `kubectl apply -f <file>.yaml`.

---

## 1. Prerequisites: Install AWS CLI on Linux
**Est. time:** 3 mins

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

---

## 2. Install AWS Provider in Crossplane: Family vs Monolithic

Crossplane provides two provider installation models:

- **Family Providers (Recommended):** Modern, modular approach. Installs a lightweight base provider plus service-specific sub-providers (e.g., `provider-aws-s3`, `provider-aws-ec2`) to conserve cluster resources and API limits.
- **Monolithic Provider (Legacy):** Installs all AWS CRDs in a single controller. Can cause cluster API slowness due to hundreds of CRDs.

**Step 1 — Create the manifest file with vi:**

```bash
vi provider-aws-s3.yaml
```

Press `i` to enter insert mode, paste/type the following, then press `Esc` and save with `:wq`:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-s3
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v1.3.0
```

**Step 2 — Apply the manifest:**

```bash
kubectl apply -f provider-aws-s3.yaml
```

**Step 3 — Verify the provider package status:**

```bash
kubectl get provider.pkg.crossplane.io
```

Ensure `HEALTHY` reports `True` and `INSTALLED` reports `True`.

---

## 3. Understand ProviderConfig Architecture: Authentication Binding

The **ProviderConfig** object serves as the operational bridge between managed resources and AWS authentication:

- It specifies **how** Crossplane authenticates (e.g., Static Access Keys, IRSA, Web Identity).
- It defines the default target AWS region.
- Individual custom resources (like an S3 Bucket) reference a specific `ProviderConfig` via `spec.providerConfigRef.name`.

---

## 4. Configure AWS Credentials Secret: Static Keys or IRSA

Choose **Option A** for quick local/testing setups or **Option B** for production EKS environments.

### Option A: Static IAM User Access Keys (Testing/Local)

**Step 1 — Create the credentials file with vi:**

```bash
vi aws-credentials.ini
```

Press `i` to enter insert mode, type the following (replace with your actual keys), then press `Esc` and save with `:wq`:

```ini
[default]
aws_access_key_id = YOUR_AWS_ACCESS_KEY_ID
aws_secret_access_key = YOUR_AWS_SECRET_ACCESS_KEY
```

**Step 2 — Store the credential file in Kubernetes as a Secret:**

```bash
kubectl create secret generic aws-secret \
  -n crossplane-system \
  --from-file=creds=./aws-credentials.ini
```

**Step 3 — Verify secret creation:**

```bash
kubectl get secret aws-secret -n crossplane-system
```

---

## 5. Apply ProviderConfig & Verify Health
**Est. time:** 2 mins

**Step 1 — Create the ProviderConfig manifest with vi:**

```bash
vi providerconfig-aws.yaml
```

Press `i` to enter insert mode, paste/type the following, then press `Esc` and save with `:wq`:

```yaml
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
```

**Step 2 — Apply the ProviderConfig:**

```bash
kubectl apply -f providerconfig-aws.yaml
```

**Step 3 — Create a test resource manifest with vi:**

```bash
vi healthcheck-bucket.yaml
```

Press `i` to enter insert mode, paste/type the following, then press `Esc` and save with `:wq`:

```yaml
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: crossplane-healthcheck-bucket
spec:
  forProvider:
    region: us-east-1
  providerConfigRef:
    name: default
```

**Step 4 — Apply the test resource:**

```bash
kubectl apply -f healthcheck-bucket.yaml
```

**Step 5 — Check reconciliation status:**

```bash
kubectl get bucket crossplane-healthcheck-bucket
```

Verify that both `READY` and `SYNCED` columns report `True`.

---

## 6. Common Provider Troubleshooting: Diagnostic Runbook

### Issue 1: CRDs Not Appearing / Stuck Package
- **Symptom:** `kubectl get bucket` returns "error: the server doesn't have a resource type".
- **Fix:** Check if package download or extraction failed due to cluster resource limits:

```bash
kubectl describe provider.pkg.crossplane.io provider-aws-s3
```

### Issue 2: Provider Pod Crash / OOMKilled
- **Symptom:** Provider pod stays in `CrashLoopBackOff` or gets killed.
- **Fix:** Monolithic providers consume >2GB RAM. Check pod logs and increase deployment memory limits:

```bash
kubectl logs -n crossplane-system -l pkg.crossplane.io/provider=provider-aws-s3
```

### Issue 3: Authentication / Unreconciled Resources
- **Symptom:** Resources stay `READY: False` with `AuthFailure` or `AccessDenied`.
- **Fix:** Verify event conditions on the resource:

```bash
kubectl describe bucket crossplane-healthcheck-bucket
```

---

## Quick Reference: vi Editor Basics

| Action | Command |
|---|---|
| Open/create a file | `vi filename.yaml` |
| Enter insert mode (to type/paste) | `i` |
| Exit insert mode | `Esc` |
| Save and quit | `:wq` then `Enter` |
| Quit without saving | `:q!` then `Enter` |
