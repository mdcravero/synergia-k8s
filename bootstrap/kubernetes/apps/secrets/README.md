# 🔐 Kubernetes Secrets Management: SOPS, Age & FluxCD

This directory contains the Kubernetes Secret definitions used by the various applications deployed in the cluster. 

Following **GitOps** principles and adopting a **Zero Trust** security model, no cryptographic material, passwords, or tokens are stored in plaintext within this repository. Secret management and versioning are securely handled via **Mozilla SOPS** (Secrets OPerationS) backed by **Age** for asymmetric encryption (AES-256-GCM).

## 🏗️ Architecture & Deployment Flow

1. **Local Encryption:** YAML manifests are encrypted on the administrator's workstation using the public Age key.
2. **Version Control:** The resulting file, containing SOPS metadata and the ciphered values, is committed and pushed to the Git repository.
3. **GitOps Synchronization (FluxCD):** The FluxCD Kustomize controller detects changes in the repository.
4. **In-Cluster Decryption:** Through the `Kustomization` resource definition (e.g., `apps-secrets`), FluxCD uses the private Age key previously provisioned in the cluster (as the `sops-age` Secret) to decrypt manifests in-memory and apply native `Secret` objects directly to the Kubernetes API.

## 🛠️ Prerequisites (Local Workstation)

To operate or modify files in this directory, the following tools must be installed and configured:
* [SOPS](https://github.com/getsops/sops) (v3.7.3 or higher)
* [Age](https://github.com/FiloSottile/age)
* The corresponding Age public/private keypair configured in the `$SOPS_AGE_KEY_FILE` environment variable or its default path.

---

## 📖 Operational Runbook

### 1. Creating a New Secret

To improve readability and avoid manual base64 encoding, using the native Kubernetes `stringData` field is highly recommended.

1. Create the plaintext manifest (e.g., `secret-app.yaml`):
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: app-postgres-secret
     namespace: default
   stringData:
     POSTGRES_URL: "postgresql://user:password@host:5432/db"