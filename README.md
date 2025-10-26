# Homelab GitOps (Flux + ESO)

This repo bootstraps a homelab Kubernetes cluster using **FluxCD** and **External Secrets Operator (ESO)**.  
All secrets come from **Azure Key Vault** via ESO — nothing sensitive is committed.

## Quick Start

### 0) Pre-create ESO access (only manual step)

```bash
kubectl create namespace external-secrets

kubectl -n external-secrets create secret generic akv-eso-creds \
  --from-literal=tenantId='<TENANT_ID>' \
  --from-literal=clientId='<ESO_SP_APP_ID>' \
  --from-literal=clientSecret='<ESO_SP_SECRET>'
```

1. Create Key Vault & populate secrets
   Use az to create a Standard SKU Key Vault and set secrets:

cert-manager DNS: azuredns-client-id, azuredns-client-secret, azuredns-subscription-id, azuredns-tenant-id

Flux vars: acme-email, azure-dns-zone, azure-client-id, azure-dns-rg, azure-subscription-id, azure-tenant-id, azure-environment, ingress-lb-ip, root-domain, wildcard-domain, tls-secret-name, kv-name

Tailscale: tailscale-client-id, tailscale-client-secret

ARC (PAT): github-token

2. Bootstrap Flux

```bash
flux bootstrap github \
  --owner <github-user-or-org> \
  --repository homelab-gitops \
  --branch main \
  --path clusters/homelab \
  --personal
```

Flux will:

Install Helm repositories

Install External Secrets Operator and connect to Key Vault

ESO creates all cluster secrets (issuer vars, DNS creds, Tailscale OAuth, ARC PAT)

Install core infra: cert-manager, MetalLB, ingress-nginx

Install operators: metrics-server, reloader, opentelemetry-operator, CNPG, Strimzi, Redis Operator

Deploy the fallbacksite with TLS for `${ROOT_DOMAIN}` and `${WILDCARD_DOMAIN}`

Verify

```bash
kubectl -n external-secrets get pods

kubectl -n cert-manager get clusterissuer homelab-issuer -o yaml | grep -A3 conditions:

kubectl -n ingress-nginx get svc ingress-nginx-controller -o wide

kubectl -n fallbacksite get ingress fallback -o yaml | grep tls: -A5
```

Notes

**Only one manual secret is created: external-secrets/akv-eso-creds**

All other secrets are created by ESO from Key Vault

To rotate credentials, update them in Key Vault; ESO will resync automatically
