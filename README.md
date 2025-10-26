# Homelab GitOps — FluxCD + External Secrets Operator

This repository bootstraps my **homelab Kubernetes cluster** using a full **GitOps** workflow powered by **FluxCD** and **External Secrets Operator (ESO)**.  
All secrets are securely managed in **Azure Key Vault**, keeping the entire cluster declarative, reproducible, and secret-free in Git.

---

## 🔧 Concept

The goal of this setup is to have **a fully automated, self-healing homelab** that syncs everything from Git — including secrets, operators, and infrastructure — without ever manually applying manifests or storing sensitive data in the repo.

Flux continuously watches this repo, applies changes automatically, and delegates secret management to ESO, which fetches them directly from Azure Key Vault.

---

## Domain & DNS Setup

I own the domain **kubesoar.com**, which I purchased through **Hostinger** and manage via **Azure DNS**.

<p align="center">
  <img src="assets/azure-dns.png" width="700" alt="Azure DNS domain configuration"/>
</p>

To enable certificate automation, I created an **Azure AD Service Principal** with the **DNS Zone Contributor** role on my DNS zone.  
This service principal is used by **cert-manager** to perform DNS-01 challenges and automatically verify ownership of my domain.

**In short:**
- Domain: `kubesoar.com` (and `*.kubesoar.com`)
- Managed in: Azure DNS  
- Certificate automation handled by: cert-manager  
- Authenticated using: Azure AD Service Principal  

---

## Secret Management — Key Vault + ESO

Secrets are not committed or templated anywhere in this repo.  
Instead, I use **Azure Key Vault** as the source of truth, and **External Secrets Operator (ESO)** pulls those secrets into Kubernetes automatically.

<p align="center">
  <img src="assets/azure-keyvault-secrets.png" width="700" alt="Azure Key Vault secrets and ESO integration"/>
</p>

This approach provides all the benefits of GitOps **without** the complexity of tools like Sealed Secrets.  
Secrets can be updated in Key Vault, and ESO will sync the updates automatically into the cluster — no re-commit or redeploy required.

**Why not Sealed Secrets?**  
They're cumbersome for secret rotation, noisy in Git history, and harder to automate.  
ESO + Key Vault is simpler, scalable, and cloud-native.

> **💡 Production Note:** If you're running this on **Azure Kubernetes Service (AKS)**, I would highly recommend using **Workload Identity** instead of service principals with client secrets. This involves creating a managed identity with a federated identity that grants a specific service account access to the managed identity in a specific namespace. You'll need OIDC and Workload Identity enabled on your AKS cluster. However, since I'm running this in my homelab via podman containers on a podman macvlan network in my homelab's LAN, I'm using service principals and client secrets for this specific setup. This isn't what I would do in AKS, but it's about all you can do in a homelab environment.

---

## Quick Start

### Step 0 — Create ESO Credentials (One-Time Manual Step)

```bash
kubectl create namespace external-secrets

kubectl -n external-secrets create secret generic akv-eso-creds \
  --from-literal=ClientID='<ESO_SP_APP_ID>' \
  --from-literal=ClientSecret='<ESO_SP_SECRET>'
```

This is the only manual secret. Everything else is fetched automatically from Azure Key Vault.

---

### Step 1 — Create & Populate Your Azure Key Vault

Use the Azure CLI to create a **Standard SKU** Key Vault and populate it with these secrets.

#### Cert-Manager DNS

* `azuredns-client-id`
* `azuredns-client-secret`
* `azuredns-subscription-id`
* `azuredns-tenant-id`

#### Flux Variables

* `acme-email`
* `azure-dns-zone`
* `azure-client-id`
* `azure-dns-rg`
* `azure-subscription-id`
* `azure-tenant-id`
* `azure-environment`
* `ingress-lb-ip`
* `root-domain`
* `wildcard-domain`
* `tls-secret-name`
* `kv-name`

#### Other Integrations

* **Tailscale**: `tailscale-client-id`, `tailscale-client-secret`
* **Actions Runner Controller (GitHub PAT)**: `github-token`

---

### Step 2 — Bootstrap Flux

```bash
flux bootstrap github \
  --owner josephaw1022 \
  --repository Homelab-Kubernetes-Cluster-Flux \
  --branch main \
  --path clusters/homelab
```

Flux will automatically:

1. Install and configure Helm repositories.
2. Deploy External Secrets Operator and connect it to Azure Key Vault.
3. Sync all required secrets into the cluster.
4. Install all core infrastructure:

   * cert-manager
   * MetalLB
   * ingress-nginx
5. Deploy operational tooling:

   * metrics-server, reloader, opentelemetry-operator, CNPG, Strimzi, Redis Operator
6. Create a TLS-secured fallback site for `${ROOT_DOMAIN}` and `${WILDCARD_DOMAIN}`.

---

### Step 3 — Verify Everything

```bash
kubectl -n external-secrets get pods

kubectl -n cert-manager get clusterissuer homelab-issuer -o yaml | grep -A3 conditions:

kubectl -n ingress-nginx get svc ingress-nginx-controller -o wide

kubectl -n fallbacksite get ingress fallback -o yaml | grep tls: -A5
```

You should see ESO pods running, a ready ClusterIssuer, and a valid ingress with a TLS section.

---

## Repository Structure

```
.
├── apps/
│   └── fallbacksite/
├── clusters/
│   └── homelab/
├── infra/
│   ├── cert-manager/
│   ├── external-secrets/
│   ├── ingress-nginx/
│   ├── kyverno/
│   ├── metallb/
│   └── ...
├── assets/
│   ├── azure-dns.png
│   └── azure-keyvault-secrets.png
└── README.md
```

---

## Notes

* Only `external-secrets/akv-eso-creds` is manually created.
* All other secrets are created dynamically by ESO.
* Secret rotation is handled automatically via Key Vault sync.
* Flux continuously reconciles all manifests and Helm releases under `clusters/homelab`.

