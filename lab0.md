# Install Kind, Create a Multi-Node Cluster, and Install Helm

Updated guide using **Kind v0.33.0** and the **kindest/node v1.37.0** image.

> Verify the latest versions before running: <https://github.com/kubernetes-sigs/kind/releases>
> Node images must be pinned with their `@sha256` digest to guarantee the image built for that Kind release.

---

## 1. Prerequisites: Install Docker (Ubuntu)

### 1.1 Add Docker's official GPG key and repository

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

### 1.2 Install Docker packages

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 1.3 Run Docker without sudo (recommended)

```bash
sudo usermod -aG docker $USER
newgrp docker
docker run --rm hello-world
```

---

## 2. Install Kind

### Step 1: Download and install the binary

```bash
#!/bin/bash
# For AMD64 / x86_64
[ "$(uname -m)" = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-linux-amd64
# For ARM64
[ "$(uname -m)" = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-linux-arm64

chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### Step 2: Verify

```bash
kind --version
```

Expected output:

```text
kind version 0.33.0
```

---

## 3. Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
kubectl version --client
```

---

## 4. Bring Up a Multi-Node Cluster

### Step 3: Create the configuration file

```bash
vi config.yaml
```

Paste the following (1 control-plane + 3 workers):

```yaml
# 4 node (3 workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  image: kindest/node:v1.37.0@sha256:a1ed56cfb0e7b93589bdf97c8cd566405a265939e3620fc4f5de89adff580ae5
- role: worker
  image: kindest/node:v1.37.0@sha256:a1ed56cfb0e7b93589bdf97c8cd566405a265939e3620fc4f5de89adff580ae5
- role: worker
  image: kindest/node:v1.37.0@sha256:a1ed56cfb0e7b93589bdf97c8cd566405a265939e3620fc4f5de89adff580ae5
- role: worker
  image: kindest/node:v1.37.0@sha256:a1ed56cfb0e7b93589bdf97c8cd566405a265939e3620fc4f5de89adff580ae5
```

Save and exit: `Esc` then `:wq`.

#### Other pre-built images for Kind v0.33.0

| Kubernetes | Image |
|------------|-------|
| v1.37.0 | `kindest/node:v1.37.0@sha256:a1ed56cfb0e7b93589bdf97c8cd566405a265939e3620fc4f5de89adff580ae5` |
| v1.36.4 | `kindest/node:v1.36.4@sha256:099e049362a1526b2db71494e1947aae99bd16290d7c895f2b7ea312e3cbfaed` |
| v1.35.8 | `kindest/node:v1.35.8@sha256:07b2536e30b803ed61d1677a79df6115f798ce64c80f9e22f6ed45afd09323c0` |
| v1.34.11 | `kindest/node:v1.34.11@sha256:44e222ee2132dab25ff87301682f89eb82c7880ea3a1bf543bfe9708fd08d67d` |

### Step 4: Start the cluster

```bash
kind create cluster --config=config.yaml
```

Sample output:

```text
Creating cluster "kind" ...
 ✓ Ensuring node image 🖼
 ✓ Preparing nodes 📦 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-kind"
```

### Step 5: Set the context

```bash
kubectl cluster-info --context kind-kind
```

---

## 5. Check Cluster Status

### Step 6: Cluster info

Using Kind:

```bash
kind get clusters
```

```text
kind
```

Using kubectl:

```bash
kubectl cluster-info
```

### Step 7: Node status

```bash
kubectl get nodes
```

```text
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   2m    v1.37.0
kind-worker          Ready    <none>          2m    v1.37.0
kind-worker2         Ready    <none>          2m    v1.37.0
kind-worker3         Ready    <none>          2m    v1.37.0
```

### Step 8: Kubernetes version

```bash
kubectl version
```

Keep the `kubectl` client within one minor version of the cluster (v1.37.x server).

---

## 6. Install Helm

The old pinned download (`helm-v3.10.2`) is outdated. Use the official installer script, which fetches the latest Helm 3 release:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
```

Or, to pin a specific version manually:

```bash
HELM_VERSION=v3.19.0   # replace with the version you want
wget https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz
tar -zxvf helm-${HELM_VERSION}-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
rm -rf linux-amd64 helm-${HELM_VERSION}-linux-amd64.tar.gz
helm version
```

---

## 7. Cleanup

```bash
kind delete cluster
```

---

## What changed from the original

| Item | Before | Now |
|------|--------|-----|
| Kind | v0.20.0 | v0.33.0 |
| Node image | `kindest/node:v1.28.0` | `kindest/node:v1.37.0` (digest-pinned) |
| Docker | root only | Added non-root `docker` group setup |
| kubectl | assumed installed | Install step added |
| Helm | v3.10.2, `mv` without sudo | Official installer script, `sudo mv` |
