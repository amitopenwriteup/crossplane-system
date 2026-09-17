# Lab: Enable Versioning, Public Access & Host a Static HTML Page

This lab extends the `crossplane-healthcheck-bucket` created earlier. You'll enable versioning, open public access, host a static website, upload an `index.html` file, and verify the page is publicly reachable.

> **IMPORTANT — Replace the bucket name:** Every YAML manifest in this lab uses `crossplane-healthcheck-bucket` as a placeholder (in `spec.forProvider.bucket`, in the policy's `Resource` ARN, and in the website URL at the end). **Replace it with your own actual bucket name in every file before applying**, or all steps will fail or target the wrong bucket.

> **Note:** `region` is a **required** field under `spec.forProvider` on every S3 sub-resource MR in this provider version — omitting it fails validation with `spec.forProvider.region: Required value`. Every manifest below already includes it (`us-east-1`); change it to match your actual bucket's region.

> **Note:** Field names below (`forProvider.*`) reflect the `provider-aws-s3` v1.x CRD schema. Before applying, confirm exact fields for your installed version with:
> ```bash
> kubectl explain bucketversioning.spec.forProvider
> kubectl explain bucketpublicaccessblock.spec.forProvider
> kubectl explain bucketwebsiteconfiguration.spec.forProvider
> kubectl explain bucketpolicy.spec.forProvider
> kubectl explain bucketobject.spec.forProvider
> ```

---

## 1. Enable Bucket Versioning

**Step 1 — Create the manifest with vi:**

```bash
vi bucketversioning.yaml
```

Press `i`, type the following, then `Esc` and `:wq`:

```yaml
apiVersion: s3.aws.upbound.io/v1beta1
kind: BucketVersioning
metadata:
  name: crossplane-healthcheck-bucket-versioning
spec:
  forProvider:
    bucket: crossplane-healthcheck-bucket   # replace with your bucket name
    region: us-east-1
    versioningConfiguration:
      - status: Enabled
  providerConfigRef:
    name: default
```

**Step 2 — Apply it:**

```bash
kubectl apply -f bucketversioning.yaml
```

**Step 3 — Verify:**

```bash
kubectl get bucketversioning crossplane-healthcheck-bucket-versioning
```

Confirm `READY` and `SYNCED` are both `True`.

---

## 2. Disable the Public Access Block (Allow Public Access)

By default, AWS blocks public access on every new bucket. To host a public website, you must explicitly turn that block off.

**Step 1 — Create the manifest with vi:**

```bash
vi bucketpublicaccessblock.yaml
```

Press `i`, type the following, then `Esc` and `:wq`:

```yaml
apiVersion: s3.aws.upbound.io/v1beta1
kind: BucketPublicAccessBlock
metadata:
  name: crossplane-healthcheck-bucket-pab
spec:
  forProvider:
    bucket: crossplane-healthcheck-bucket   # replace with your bucket name
    region: us-east-1
    blockPublicAcls: false
    blockPublicPolicy: false
    ignorePublicAcls: false
    restrictPublicBuckets: false
  providerConfigRef:
    name: default
```

**Step 2 — Apply it:**

```bash
kubectl apply -f bucketpublicaccessblock.yaml
```

**Step 3 — Verify:**

```bash
kubectl get bucketpublicaccessblock crossplane-healthcheck-bucket-pab
```

---

## 3. Add a Bucket Policy for Public Read

**Step 1 — Create the manifest with vi:**

```bash
vi bucketpolicy.yaml
```

Press `i`, type the following, then `Esc` and `:wq`:

```yaml
apiVersion: s3.aws.upbound.io/v1beta1
kind: BucketPolicy
metadata:
  name: crossplane-healthcheck-bucket-policy
spec:
  forProvider:
    bucket: crossplane-healthcheck-bucket   # replace with your bucket name
    region: us-east-1
    policy: |
      {
        "Version": "2012-10-17",
        "Statement": [
          {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::crossplane-healthcheck-bucket/*"
          }
        ]
      }
  providerConfigRef:
    name: default
```

> **Replace the bucket name in TWO places here**: both `spec.forProvider.bucket` and inside the policy JSON's `Resource` ARN (`arn:aws:s3:::<your-bucket-name>/*`). If they don't match, the policy won't apply correctly.

**Step 2 — Apply it:**

```bash
kubectl apply -f bucketpolicy.yaml
```

**Step 3 — Verify:**

```bash
kubectl get bucketpolicy crossplane-healthcheck-bucket-policy
```

---

## 4. Enable Static Website Hosting

**Step 1 — Create the manifest with vi:**

```bash
vi bucketwebsiteconfiguration.yaml
```

Press `i`, type the following, then `Esc` and `:wq`:

```yaml
apiVersion: s3.aws.upbound.io/v1beta1
kind: BucketWebsiteConfiguration
metadata:
  name: crossplane-healthcheck-bucket-website
spec:
  forProvider:
    bucket: crossplane-healthcheck-bucket   # replace with your bucket name
    region: us-east-1
    indexDocument:
      - suffix: index.html
  providerConfigRef:
    name: default
```

**Step 2 — Apply it:**

```bash
kubectl apply -f bucketwebsiteconfiguration.yaml
```

**Step 3 — Verify:**

```bash
kubectl get bucketwebsiteconfiguration crossplane-healthcheck-bucket-website
```

---

## 5. Create a Simple HTML Page

**Step 1 — Create `index.html` with vi:**

```bash
vi index.html
```

Press `i`, type the following, then `Esc` and `:wq`:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Crossplane Demo Page</title>
  </head>
  <body>
    <h1>Hello from Crossplane!</h1>
    <p>This page is hosted on an S3 bucket provisioned entirely by Crossplane.</p>
  </body>
</html>
```

---

## 6. Upload the HTML File to the Bucket

You can upload the file either declaratively (via Crossplane) or directly (via AWS CLI). Pick one.

### Option A: Declarative Upload with Crossplane (BucketObject)

**Step 1 — Create the manifest with vi:**

```bash
vi bucketobject.yaml
```

Press `i`, type the following, then `Esc` and `:wq`. Paste the HTML content directly under `content:` as plain text (no encoding needed):

```yaml
apiVersion: s3.aws.upbound.io/v1beta1
kind: BucketObject
metadata:
  name: crossplane-healthcheck-bucket-index
spec:
  forProvider:
    bucket: crossplane-healthcheck-bucket   # replace with your bucket name
    region: us-east-1
    key: index.html
    contentType: text/html
    content: |
      <!DOCTYPE html>
      <html>
        <head>
          <title>Crossplane Demo Page</title>
        </head>
        <body>
          <h1>Hello from Crossplane!</h1>
          <p>This page is hosted on an S3 bucket provisioned entirely by Crossplane.</p>
        </body>
      </html>
  providerConfigRef:
    name: default
```

**Step 2 — Apply it:**

```bash
kubectl apply -f bucketobject.yaml
```

**Step 3 — Verify:**

```bash
kubectl get bucketobject crossplane-healthcheck-bucket-index
```

---

## 7. Get the Website URL via kubectl

Crossplane stores the AWS-computed website endpoint in the resource's `status.atProvider` field — no AWS CLI needed.

**Step 1 — Inspect the full status to find the endpoint field:**

```bash
kubectl get bucketwebsiteconfiguration crossplane-healthcheck-bucket-website -o yaml
```

Look under `status.atProvider` for a field such as `websiteEndpoint` (the exact key name can vary slightly by provider version).

**Step 2 — Print just the endpoint directly:**

```bash
kubectl get bucketwebsiteconfiguration crossplane-healthcheck-bucket-website -o jsonpath='{.status.atProvider.websiteEndpoint}'
```

> If this returns empty, the field name differs in your provider version — check the full `-o yaml` output from Step 1 and adjust the `jsonpath` key accordingly (e.g. it may be nested differently, such as `.status.atProvider.websiteDomain`).

**Step 3 — Build the full URL from the endpoint:**

```bash
echo "http://$(kubectl get bucketwebsiteconfiguration crossplane-healthcheck-bucket-website -o jsonpath='{.status.atProvider.websiteEndpoint}')"
```

**Step 4 — Test it:**

```bash
curl -I "http://$(kubectl get bucketwebsiteconfiguration crossplane-healthcheck-bucket-website -o jsonpath='{.status.atProvider.websiteEndpoint}')"
```

Expect `HTTP/1.1 200 OK`. Open the printed URL in a browser to see the "Hello from Crossplane!" page.

---

## 8. Cleanup (Optional)

To tear down what this lab created, in reverse order:

```bash
kubectl delete -f bucketobject.yaml
kubectl delete -f bucketwebsiteconfiguration.yaml
kubectl delete -f bucketpolicy.yaml
kubectl delete -f bucketpublicaccessblock.yaml
kubectl delete -f bucketversioning.yaml
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
