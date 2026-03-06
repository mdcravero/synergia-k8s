# 🔐 Kubernetes Secrets Management: SOPS, Age & FluxCD

This directory contains the Kubernetes Secret definitions used by the various applications deployed in the cluster.

Following **GitOps** principles and adopting a **Zero Trust** security model, no cryptographic material, passwords, or tokens are stored in plaintext within this repository. Secret management and versioning are securely handled via **Mozilla SOPS** (Secrets OPerationS) backed by **Age** for asymmetric encryption (AES-256-GCM).

## 🏗️ Architecture & Deployment Flow

1. **Local Encryption:** YAML manifests are encrypted on the administrator's workstation using the public Age key.
2. **Version Control:** The resulting file, containing SOPS metadata and the ciphered values, is committed and pushed to the Git repository.
3. **GitOps Synchronization (FluxCD):** The FluxCD Kustomize controller detects changes in the repository.
4. **In-Cluster Decryption:** Through the `Kustomization` resource definition (e.g., `apps-secrets`), FluxCD uses the private Age key previously provisioned in the cluster (as the `sops-age` Secret) to decrypt manifests in-memory and apply native `Secret` objects directly to the Kubernetes API.

---

## ⚙️ Workstation Setup (Debian / Ubuntu)

To operate or modify files in this directory, you need to set up your local environment. Run the following commands to install the required dependencies and generate your Age keypair.

### 1. Install Age

Age is available in the standard APT repositories:

```bash
sudo apt update
sudo apt install age -y
```

### 2. Install SOPS

Download and install the latest SOPS release (check the GitHub Releases page for the newest version):

```bash
curl -LO https://github.com/getsops/sops/releases/download/v3.8.1/sops_3.8.1_amd64.deb
sudo dpkg -i sops_3.8.1_amd64.deb
```

### 3. Generate Age Keypair

Create a secure directory and generate your private/public keypair:

```bash
mkdir -p ~/.config/sops/age
age-keygen -o ~/.config/sops/age/keys.txt
```

> ⚠️ **CRITICAL:** Keep this keys.txt file safe and NEVER commit it to the repository. The public key will be printed in the terminal output.

### 4. Configure Environment

Tell SOPS where to find your private key by exporting the environment variable:

```bash
echo 'export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt' >> ~/.bashrc
source ~/.bashrc
```

---

## 📖 Operational Runbook

### 1. Creating a New Secret

To improve readability and avoid manual base64 encoding, using the native Kubernetes stringData field is highly recommended.

Create the plaintext manifest (e.g., secret-app.yaml):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-postgres-secret
  namespace: default
stringData:
  POSTGRES_URL: "postgresql://user:password@host:5432/db"
```

Execute in-place encryption (the original file will be replaced by its encrypted version):

```bash
sops --encrypt --in-place secret-app.yaml
```

Verify that the sensitive values are wrapped under the `ENC[AES256_GCM...]` directive and commit the file.

### 2. Editing an Existing Secret

> ⚠️ **IMPORTANT:** Never manually modify the ciphered text or the SOPS metadata directly within the .yaml file. Doing so will invalidate the Message Authentication Code (MAC) and cause FluxCD's integrity validation to fail.

To securely update a value, use the SOPS edit command. This will decrypt the file into a temporary in-memory buffer (RAM) and open it in your default text editor:

```bash
# Edit using the system's default editor
sops secret-app.yaml

# Alternative: Force a specific editor (e.g., nano or vim)
EDITOR=nano sops secret-app.yaml
```

Upon saving and closing the editor, SOPS will automatically recalculate the hashes and re-encrypt the file before writing it to disk.

### 3. Local Viewing & Auditing

To inspect the contents of an encrypted secret without the risk of altering its cryptographic signatures, output the decrypted content directly to standard output (stdout):

```bash
sops -d secret-app.yaml
```

---

## 🚀 FluxCD Integration (Reference)

For the cluster to successfully interpret these files, the Kustomization resource responsible for applying this directory must include the decryption provider pointing to the in-cluster Secret containing the Age private key.

Example of the required controller configuration:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps-secrets
  namespace: flux-system
spec:
  path: ./bootstrap/kubernetes/apps/secrets
  decryption:
    provider: sops
    secretRef:
      name: sops-age # Name of the in-cluster secret holding the private key
```
