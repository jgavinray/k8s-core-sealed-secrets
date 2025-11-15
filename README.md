# k8s-core-sealed-secrets

A Helm chart for deploying the Bitnami Sealed Secrets Controller.

## Description

This Helm chart wraps the Bitnami Sealed Secrets Controller chart as a dependency. **This chart is designed to be deployed via ArgoCD.**

The Sealed Secrets Controller enables you to store encrypted secrets in Git repositories safely. The controller decrypts these sealed secrets and creates regular Kubernetes secrets in your cluster.

### Architecture and Key Management

**Sealed Secrets is a singleton application** - typically one controller instance per cluster manages all sealed secrets. The controller generates an RSA key pair on first startup:

- **Default Key Size**: 4096-bit RSA (considered secure for most use cases)
- **Supported Key Sizes**: 2048, 3072, 4096, 8192 bits (configurable via `args` in values.yaml)
- **Public Key**: Used by `kubeseal` to encrypt secrets. Can be safely shared.
- **Private Key**: Stored in a Kubernetes Secret (default: `sealed-secrets-key`) and used by the controller to decrypt sealed secrets.

**Note**: To use a stronger RSA key (e.g., 8192 bits), configure `args` in `values.yaml` with the `--key-size` flag before the initial deployment. The key size cannot be changed after the key pair is generated.

Example for 8192-bit RSA key:
```yaml
sealed-secrets:
  args: ["--key-size", "8192", "--update-status"]
```

**Important**: When setting `args`, you must include all desired flags (e.g., `--update-status`, `--skip-recreate`) as `args` overrides the default arguments.

**Critical**: If the private key is lost, you **cannot decrypt existing sealed secrets**. The controller pod can be recreated, but the private key Secret must be preserved.

To retrieve and back up the private key Secret:

```bash
# Replace <namespace> with the namespace where sealed-secrets is deployed
kubectl get secret sealed-secrets-key -n <namespace> -o yaml > sealed-secrets-key-backup.yaml
```

**Recommendation**: Store the private key backup securely in a password manager or another secure secret management system. Never commit the private key to Git repositories.

**Important Considerations**:
- The controller is cluster-wide by default - all namespaces use the same key pair
- If the controller pod is lost but the Secret remains, you can recover by redeploying
- If the Secret with the private key is lost, all existing sealed secrets become unrecoverable
- Always back up the private key Secret (see Backup and Recovery section below)

All configuration options are documented inline in the `values.yaml` file.

## Table of Contents

- [Managing Secrets](#managing-secrets)
  - [Prerequisites](#prerequisites)
  - [Adding a New Secret](#adding-a-new-secret)
  - [Updating an Existing Secret](#updating-an-existing-secret)
  - [Removing a Secret](#removing-a-secret)
  - [Working with Multiple Namespaces](#working-with-multiple-namespaces)
- [Persisting Secrets to GitHub](#persisting-secrets-to-github)
- [Getting the Public Key](#getting-the-public-key)
- [Multiple Clusters/Environments](#multiple-clustersenvironments)
  - [Backing Up Private Keys](#backing-up-private-keys)
  - [Managing Public Keys](#managing-public-keys)
- [Backup and Recovery](#backup-and-recovery)
  - [Backing Up the Private Key](#backing-up-the-private-key)
  - [Restoring the Private Key](#restoring-the-private-key)
  - [Disaster Recovery Scenarios](#disaster-recovery-scenarios)

## Managing Secrets

Once the Sealed Secrets Controller is deployed, you can manage secrets using the `kubeseal` CLI tool.

### Prerequisites

Install the `kubeseal` CLI tool:

```bash
# macOS
brew install kubeseal

# Linux
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.29.0/kubeseal-0.29.0-linux-amd64.tar.gz
tar -xvzf kubeseal-0.29.0-linux-amd64.tar.gz kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal

# Or download from: https://github.com/bitnami-labs/sealed-secrets/releases
```

### Adding a New Secret

1. **Create a Kubernetes Secret YAML file** (avoid using `kubectl create secret` as it exposes values in bash history):

```yaml
# my-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
type: Opaque
data:
  username: YWRtaW4=  # base64 encoded: admin
  password: c2VjcmV0MTIz  # base64 encoded: secret123
```

**Note**: Values must be base64 encoded. You can encode values using:
```bash
echo -n "your-value" | base64
```

2. **Seal the secret**:

```bash
kubeseal < my-secret.yaml > my-sealed-secret.yaml
```

3. **Delete the unsealed secret file** (for security):

```bash
rm my-secret.yaml
```

4. **Commit to Git**:

```bash
git add my-sealed-secret.yaml
git commit -m "Add sealed secret for my-secret"
git push
```

ArgoCD will automatically apply the `SealedSecret` resource, and the controller will decrypt it and create the Kubernetes `Secret`.

### Updating an Existing Secret

You can update a secret in two ways:

#### Method 1: Create New Secret (Recommended)

1. **Create a new secret YAML file** with updated values:

```yaml
# my-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
type: Opaque
data:
  username: bmV3YWRtaW4=  # base64 encoded: newadmin
  password: bmV3c2VjcmV0NDU2  # base64 encoded: newsecret456
```

2. **Reseal the secret**:

```bash
kubeseal < my-secret.yaml > my-sealed-secret.yaml
```

3. **Delete the unsealed secret file** (for security):

```bash
rm my-secret.yaml
```

4. **Commit and push**:

```bash
git add my-sealed-secret.yaml
git commit -m "Update sealed secret for my-secret"
git push
```

#### Method 2: Unseal Existing Secret for Rotation

If you need to decrypt an existing sealed secret to rotate it:

1. **Retrieve the private key** from the cluster:

```bash
# Replace <namespace> with the namespace where sealed-secrets is deployed
kubectl get secret sealed-secrets-key -n <namespace> -o yaml > sealed-secrets-key-backup.yaml
```

2. **Unseal the existing sealed secret**:

```bash
kubeseal --recovery-unseal --recovery-private-key sealed-secrets-key-backup.yaml < my-sealed-secret.yaml > my-secret.yaml
```

3. **Update the values** in `my-secret.yaml` (edit the base64 encoded values)

4. **Reseal the updated secret**:

```bash
kubeseal < my-secret.yaml > my-sealed-secret.yaml
```

5. **Clean up sensitive files**:

```bash
rm my-secret.yaml sealed-secrets-key-backup.yaml
```

6. **Commit and push**:

```bash
git add my-sealed-secret.yaml
git commit -m "Rotate sealed secret for my-secret"
git push
```

**Security Note**: The private key allows decryption of all sealed secrets. Handle it carefully and delete the backup file immediately after use.

### Removing a Secret

1. **Delete the sealed secret file**:

```bash
git rm my-sealed-secret.yaml
git commit -m "Remove sealed secret for my-secret"
git push
```

ArgoCD will remove the `SealedSecret` resource from the cluster. The controller may automatically delete the underlying Kubernetes secret depending on your configuration.

### Working with Multiple Namespaces

Specify the namespace when sealing:

```bash
kubeseal -n my-namespace < my-secret.yaml > my-sealed-secret.yaml
```

## Persisting Secrets to GitHub

Sealed secrets are encrypted using the controller's public key and can be safely stored in Git:

- **Encrypted at Rest**: Secret values cannot be decrypted without the controller's private key (which only exists in the cluster)
- **Git-Friendly**: Regular YAML files that can be version controlled, reviewed in PRs, and tracked for changes
- **Best Practices**:
  - Store sealed secrets in the same repository as your application manifests
  - Use descriptive filenames (e.g., `database-sealed-secret.yaml`)
  - Never commit unsealed (plain) Kubernetes secrets to Git - always delete the unsealed YAML file after sealing
  - Regularly rotate secrets by resealing them with new values

## Getting the Public Key

To seal secrets from a machine without direct cluster access:

```bash
# Get the public key from the controller
kubeseal --fetch-cert > public-key.pem

# Use the public key to seal secrets offline
kubeseal --cert=public-key.pem < my-secret.yaml > my-sealed-secret.yaml
```

The public key can be safely committed to your repository root (not in `templates/`), as it can only encrypt secrets (not decrypt them).

## Multiple Clusters/Environments

If you have multiple clusters (e.g., dev, staging, production), each cluster has its own sealed-secrets controller with a unique key pair. You need separate public keys for each cluster.

### Backing Up Private Keys

Back up each private key with environment-specific naming and store in a password manager:

```bash
# Development cluster
kubectl config use-context dev
kubectl get secret sealed-secrets-key -n <namespace> -o yaml > sealed-secrets-key-dev.yaml

# Staging cluster
kubectl config use-context staging
kubectl get secret sealed-secrets-key -n <namespace> -o yaml > sealed-secrets-key-staging.yaml

# Production cluster
kubectl config use-context prod
kubectl get secret sealed-secrets-key -n <namespace> -o yaml > sealed-secrets-key-prod.yaml
```

Store each in your password manager with labels: `sealed-secrets-key-dev`, `sealed-secrets-key-staging`, `sealed-secrets-key-prod`

### Managing Public Keys

Fetch and commit environment-specific public keys to your repository root (not in `templates/`):

```bash
# Development
kubectl config use-context dev
kubeseal --fetch-cert > public-key-dev.pem

# Staging
kubectl config use-context staging
kubeseal --fetch-cert > public-key-staging.pem

# Production
kubectl config use-context prod
kubeseal --fetch-cert > public-key-prod.pem
```

Commit these files to your repository root. Use them to seal secrets for specific environments:

```bash
# Seal for development
kubeseal --cert=public-key-dev.pem < my-secret.yaml > my-sealed-secret-dev.yaml

# Seal for production
kubeseal --cert=public-key-prod.pem < my-secret.yaml > my-sealed-secret-prod.yaml
```

## Backup and Recovery

### Backing Up the Private Key

**Critical**: The private key must be backed up to prevent permanent loss of access to sealed secrets.

1. **Export the private key Secret**:

```bash
# Replace <namespace> with the namespace where sealed-secrets is deployed
kubectl get secret sealed-secrets-key -n <namespace> -o yaml > sealed-secrets-key-backup.yaml
```

2. **Store the backup securely**:
   - Store in a secure secret management system (e.g., HashiCorp Vault, AWS Secrets Manager)
   - Encrypt the backup file before storing
   - Never commit the private key to Git repositories

### Restoring the Private Key

If you need to restore the controller with the same key (e.g., after cluster migration or disaster recovery):

1. **Apply the backed-up Secret**:

```bash
kubectl apply -f sealed-secrets-key-backup.yaml
```

2. **Restart the controller** to pick up the restored key:

```bash
kubectl delete pod -n <namespace> -l app.kubernetes.io/name=sealed-secrets
```

The controller will use the restored private key and can decrypt all previously sealed secrets.

### Disaster Recovery Scenarios

- **Controller pod lost, Secret intact**: Simply redeploy the controller - the Secret persists and recovery is automatic
- **Secret lost, sealed secrets in Git**: You cannot decrypt existing sealed secrets. You must:
  1. Deploy a new controller (generates a new key pair)
  2. Reseal all secrets with the new public key
  3. Update all sealed secret files in Git
- **Full cluster loss**: Restore the private key Secret from backup before deploying the controller, or all sealed secrets must be resealed
