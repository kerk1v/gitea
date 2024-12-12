# Gitea Kubernetes Deployment

This repository contains Kubernetes manifests for deploying Gitea with MariaDB using ArgoCD on a K3s cluster.

## Prerequisites

- K3s cluster
- ArgoCD installed
- Traefik ingress controller
- cert-manager with ClusterIssuer "letsencrypt" configured
- kubectl configured to access your cluster
- Port 2222 available on the Kubernetes nodes for SSH access

## Project Structure

```
.
├── base
│   ├── gitea/           # Gitea base configuration
│   │   ├── configmap.yaml
│   │   ├── deployment.yaml
│   │   ├── ingress.yaml
│   │   ├── kustomization.yaml
│   │   ├── pvc.yaml
│   │   └── service.yaml
│   └── mariadb/         # MariaDB base configuration
│       ├── deployment.yaml
│       ├── kustomization.yaml
│       ├── pvc.yaml
│       └── service.yaml
└── overlays
    └── production/      # Production environment specific configuration
        ├── kustomization.yaml
        └── namespace.yaml
```

## Secret Configuration

Before deploying, you need to create a secret in the gitea namespace containing the database credentials. Here's how to set it up:

1. First, create the namespace:
```bash
kubectl create namespace gitea
```

2. Create the secret with all required keys:
```bash
kubectl create secret generic gitea-db-secrets \
  --namespace gitea \
  --from-literal=mariadb-root-password='your-root-password' \
  --from-literal=mariadb-password='your-gitea-password' \
  --from-literal=security-key='random-32-char-string' \
  --from-literal=internal-token='random-64-char-string' \
  --from-literal=jwt-secret='random-32-char-string' \
  --from-literal=lfs-jwt-secret='random-32-char-string'
```

Required secret keys:
- `mariadb-root-password`: Root password for MariaDB
- `mariadb-password`: Password for Gitea database user
- `security-key`: Used for signing sessions, should be 32 characters
- `internal-token`: Used for internal API calls, should be 64 characters
- `jwt-secret`: Used for JWT token generation, should be 32 characters
- `lfs-jwt-secret`: Used for Git LFS authentication, should be 32 characters

You can generate random strings for the secrets using:
```bash
# For 32-character strings (security-key, jwt-secret, lfs-jwt-secret)
openssl rand -base64 24

# For 64-character string (internal-token)
openssl rand -base64 48
```

## Deployment

1. Apply the ArgoCD application:
```bash
kubectl apply -f gitea-application.yaml
```

2. Verify the application sync status:
```bash
argocd app get gitea
```

## Configuration Details

### Storage
- Both Gitea and MariaDB use persistent volumes with the local-path storage class
- Gitea data: 20Gi
- MariaDB data: 10Gi

### Networking
- HTTP: Accessible via HTTPS at git.conxtor.com (through Traefik Ingress)
- SSH: Directly exposed on port 2222 on all Kubernetes nodes (NodePort)
- TLS: Enabled via cert-manager with Let's Encrypt for HTTPS

### Git SSH Access
SSH access is configured on port 2222 and is exposed directly on the Kubernetes nodes. To use Git over SSH:

1. Add to your ~/.ssh/config:
```
Host git.conxtor.com
    Port 2222
    User git
```

2. Clone repositories using:
```bash
git clone git@git.conxtor.com:username/repository.git
```

### Database
- MariaDB 10.11
- Database name: gitea
- User: gitea
- Passwords: Stored in pre-created Kubernetes secret

### Resources
MariaDB:
- Requests: 250m CPU, 256Mi memory
- Limits: 500m CPU, 512Mi memory

Gitea:
- Requests: 250m CPU, 256Mi memory
- Limits: 1000m CPU, 1Gi memory

## Security Notes

- All sensitive information is stored in Kubernetes secrets
- The secret must be created before deploying the application
- Each secret key serves a specific security purpose
- TLS is enabled by default using Let's Encrypt certificates
- SSH access is configured on a non-standard port (2222) for better security

## Troubleshooting

### ArgoCD Sync Issues
If you see sync issues:
1. Check if the application is created properly:
```bash
kubectl get application gitea -n argocd
```

2. Check the application status:
```bash
argocd app get gitea
```

3. View detailed sync logs:
```bash
argocd app logs gitea
```

## Maintenance

### Updating Secrets
To update the secrets:

1. Create a new secret manifest:
```bash
kubectl create secret generic gitea-db-secrets \
  --namespace gitea \
  --from-literal=mariadb-root-password='new-root-password' \
  --from-literal=mariadb-password='new-gitea-password' \
  --from-literal=security-key='new-security-key' \
  --from-literal=internal-token='new-internal-token' \
  --from-literal=jwt-secret='new-jwt-secret' \
  --from-literal=lfs-jwt-secret='new-lfs-jwt-secret' \
  --dry-run=client -o yaml | kubectl apply -f -
```

2. Restart the affected pods:
```bash
kubectl rollout restart deployment mariadb -n gitea
kubectl rollout restart deployment gitea -n gitea
```

### Verifying Deployment
To verify the deployment:

1. Check pod status:
```bash
kubectl get pods -n gitea
```

2. Check logs:
```bash
kubectl logs -f deployment/gitea -n gitea
kubectl logs -f deployment/mariadb -n gitea
```

3. Verify services:
```bash
# Check HTTP service
curl -I https://git.conxtor.com

# Check SSH service
ssh -T -p 2222 git@git.conxtor.com
```

### Firewall Configuration
Ensure that port 2222 is open on your Kubernetes nodes for SSH access. You may need to configure your firewall rules accordingly:

```bash
# Example for UFW
sudo ufw allow 2222/tcp

# Example for firewalld
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --reload