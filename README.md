# 🔐 Certificate Management

Automated TLS certificate management for MutFarm homelab using cert-manager with Let's Encrypt.

## Overview

- **Provider:** Let's Encrypt
- **Challenge:** DNS-01 via Cloudflare
- **Certificate:** Wildcard `*.enricocc.com` + root domain
- **Auto-renewal:** Yes (cert-manager handles it)
- **Distribution:** Stakater Replicator copies to multiple namespaces

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ cert-manager (namespace: cert-manager)                      │
│                                                             │
│  ClusterIssuer (Cloudflare DNS-01)                         │
│         ↓                                                   │
│  Certificate wildcard-enricocc-tls                         │
│         ↓                                                   │
│  Secret: wildcard-enricocc-tls (with replicator annotation)│
└─────────────────────────────────────────────────────────────┘
         ↓ (Stakater Replicator)
┌────────────────────────────────────────────────────────────┐
│ Replicated to namespaces:                                 │
│  - adguard                                                │
│  - cattle-system (Rancher)                                │
│  - longhorn-system                                        │
└────────────────────────────────────────────────────────────┘
```

## Quick Start

### 1. Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

### 2. Install Stakater Replicator

```bash
helm repo add mittwald https://helm.mittwald.de && helm repo update
helm upgrade --install kubernetes-replicator mittwald/kubernetes-replicator \
  --namespace replicator --create-namespace
```

### 3. Create Cloudflare API Token Secret

```bash
# Create from template
cp cert-manager/secret.yaml.example cert-manager/secret.yaml

# Edit with your Cloudflare API token
vim cert-manager/secret.yaml

# Apply
kubectl apply -f cert-manager/secret.yaml
```

### 4. Apply Certificate Configuration

```bash
# ClusterIssuer
kubectl apply -f cert-manager/cloudflaredn01-clusterissuer.yaml

# Wildcard Certificate (auto-renews every 60 days)
kubectl apply -f cert-manager/wildcard-certificate.yaml
```

## Files

```
cert-manager/
├── README.md                          # This file
├── cert-manager.yaml                  # cert-manager installation manifest
├── cloudflaredn01-clusterissuer.yaml  # Cloudflare DNS-01 issuer
├── letsencrypt-clusterissuer.yaml     # Alternative HTTP-01 issuer
├── wildcard-certificate.yaml          # Wildcard cert config
├── secret.yaml.example                # Template for Cloudflare token
└── .gitignore                         # Protects secret.yaml
```

## How It Works

### Certificate Renewal (Automatic)

cert-manager checks certificates **30 days before expiry** and automatically renews them:

1. cert-manager detects cert is expiring soon
2. Creates DNS TXT record in Cloudflare: `_acme-challenge.enricocc.com`
3. Let's Encrypt verifies the challenge
4. New certificate is issued and stored in the secret
5. Replicator automatically updates all target namespaces
6. Services using the cert get the new version without restart

**You don't need to do anything!** Just monitor:

```bash
# Check certificate status
kubectl -n cert-manager get certificate wildcard-enricocc-tls

# Check cert-manager logs
kubectl -n cert-manager logs -l app=cert-manager -f

# Check expiry date
kubectl -n cert-manager get secret wildcard-enricocc-tls -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -enddate
```

### Adding a New Namespace

**Option 1: Update Certificate (recommended)**

Edit `wildcard-certificate.yaml`:

```yaml
secretTemplate:
  annotations:
    replicator.v1.mittwald.de/replicate-to: "adguard,cattle-system,longhorn-system,NEW-NAMESPACE"
```

Apply:
```bash
kubectl apply -f cert-manager/wildcard-certificate.yaml
```

**Option 2: Annotate existing secret**

```bash
kubectl -n cert-manager annotate secret wildcard-enricocc-tls \
  replicator.v1.mittwald.de/replicate-to="adguard,cattle-system,longhorn-system,NEW-NAMESPACE" \
  --overwrite
```

**Option 3: Pull-based (in target namespace)**

Create a secret that pulls from source:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: wildcard-enricocc-tls
  namespace: NEW-NAMESPACE
  annotations:
    replicator.v1.mittwald.de/replicate-from: cert-manager/wildcard-enricocc-tls
type: kubernetes.io/tls
data:
  tls.crt: ""
  tls.key: ""
```

### Using the Certificate in Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: my-namespace
spec:
  tls:
  - hosts:
    - service.enricocc.com
    secretName: wildcard-enricocc-tls  # The replicated secret
  rules:
  - host: service.enricocc.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

## Troubleshooting

### Certificate not issuing

```bash
# Check certificate status
kubectl -n cert-manager describe certificate wildcard-enricocc-tls

# Check certificate request
kubectl -n cert-manager get certificaterequest

# Check challenges
kubectl -n cert-manager get challenge

# Detailed logs
kubectl -n cert-manager logs -l app=cert-manager -f
```

Common issues:
- **Cloudflare token invalid**: Check secret `cloudflare-api-token`
- **DNS propagation**: Challenge may take 1-2 minutes
- **Rate limits**: Let's Encrypt has rate limits (50 certs/domain/week)

### Secret not replicating

```bash
# Check replicator logs
kubectl -n replicator logs -l app.kubernetes.io/name=kubernetes-replicator -f

# Check source secret has annotation
kubectl -n cert-manager get secret wildcard-enricocc-tls -o yaml | grep replicate

# Check RBAC permissions
kubectl -n replicator get serviceaccount
kubectl get clusterrole | grep replicator
```

### Ingress not using certificate

```bash
# Verify secret exists in namespace
kubectl -n YOUR-NAMESPACE get secret wildcard-enricocc-tls

# Check ingress configuration
kubectl -n YOUR-NAMESPACE describe ingress YOUR-INGRESS

# Check ingress-nginx logs
kubectl -n ingress-nginx logs -l app.kubernetes.io/name=ingress-nginx -f | grep tls
```

## Manual Operations (rarely needed)

### Force certificate renewal

```bash
# Delete the secret (cert-manager will recreate)
kubectl -n cert-manager delete secret wildcard-enricocc-tls

# Or annotate certificate to force renewal
kubectl -n cert-manager annotate certificate wildcard-enricocc-tls \
  cert-manager.io/issue-temporary-certificate=true --overwrite
```

### Check certificate details

```bash
# Decode and view certificate
kubectl -n cert-manager get secret wildcard-enricocc-tls -o jsonpath='{.data.tls\.crt}' | \
  base64 -d | openssl x509 -noout -text
```

### Backup certificates

```bash
# Export certificate and key
kubectl -n cert-manager get secret wildcard-enricocc-tls -o yaml > cert-backup-$(date +%Y%m%d).yaml
```

## Migration from certbot (deprecated)

If you were using certbot before, the old manual process is **no longer needed**. 

The old `cert-bot-renew/` directory and manual certbot commands are archived. cert-manager handles everything automatically now.

Old certbot files in `/etc/letsencrypt/` and `certs/` can be removed once cert-manager is working properly.

## Cloudflare API Token Setup

Required permissions for the token:
- **Zone → DNS → Edit**
- **Zone → Zone → Read**

Scoped to zone: `enricocc.com`

Create at: https://dash.cloudflare.com/profile/api-tokens

## Useful Commands

```bash
# Watch certificate status
watch kubectl -n cert-manager get certificate

# Check all secrets across namespaces
kubectl get secrets --all-namespaces | grep wildcard-enricocc-tls

# View replicator config
kubectl -n cert-manager get certificate wildcard-enricocc-tls -o yaml

# Manually trigger sync (delete target to force re-replication)
kubectl -n TARGET-NS delete secret wildcard-enricocc-tls
```
