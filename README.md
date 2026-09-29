# 🔒 Meridian Site: Azure Static Web App with Passwordless OIDC

> Enterprise-grade web delivery on Azure Static Web Apps, using GitHub OIDC Workload Identity Federation for zero-secret CI/CD and Microsoft Entra ID for RBAC-scoped access.

---

## 🏗️ Architecture & Overview

```text
[Developer Push] ➔ [GitHub Actions Workflow]
        │  (OIDC federated auth / short-lived token)
        ▼
[Azure Resource Manager]
        │
        ▼
[Azure Static Web App] ◄──► [Microsoft Entra ID]
                            (RBAC / Identity Management)
```

- **Problem:** Typical CI/CD pipelines rely on long-lived Azure service principal client secrets or deployment tokens stored in repository secrets. These create rotation overhead and a serious credential-exposure risk.
- **Solution:** OpenID Connect (OIDC) identity federation between GitHub Actions and Microsoft Entra ID. The pipeline requests a short-lived JSON Web Token (JWT) on demand to deploy to Azure Static Web Apps, so no persistent cloud credentials are stored in GitHub.

---

## 🛠️ Tech Stack

![Azure](https://img.shields.io/badge/azure-%230072C6.svg?style=flat&logo=microsoftazure&logoColor=white)
![Entra ID](https://img.shields.io/badge/entra%20id-%230078D4.svg?style=flat&logo=microsoftazure&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/github%20actions-%232088FF.svg?style=flat&logo=githubactions&logoColor=white)
![OIDC](https://img.shields.io/badge/auth-OIDC%20Federation-green.svg?style=flat)

---

## 🔑 Key Engineering Features

- **Passwordless authentication (OIDC):** Federated identity credentials scoped to specific repository subjects (the `main` branch and pull requests), eliminating static API keys and long-lived client secrets.
- **Least-privilege RBAC:** The Entra ID app registration is granted `Contributor` scoped only to the Static Web App's resource group.
- **Automated production and staging workflows:** Pull requests create isolated preview environments; merges to `main` deploy to production and clean up staging.

---

## 🚀 Getting Started & Deployment

### Prerequisites

- Azure CLI installed locally (`az login`)
- An Azure subscription with permission to create app registrations and role assignments

### 1. Entra ID & OIDC setup

```bash
# Create the app registration and its service principal
az ad app create --display-name "github-actions-meridian-swa"
az ad sp create --id <APP_ID>

# Federated credential for the main branch
az ad app federated-credential create --id <APP_OBJECT_ID> --parameters '{
  "name": "github-actions-main-branch",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:YOUR_GITHUB_USERNAME/meridian-site:ref:refs/heads/main",
  "description": "OIDC federation for Meridian Site main branch",
  "audiences": ["api://AzureADTokenExchange"]
}'

# Federated credential for pull request preview deployments
az ad app federated-credential create --id <APP_OBJECT_ID> --parameters '{
  "name": "github-actions-pull-requests",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:YOUR_GITHUB_USERNAME/meridian-site:pull_request",
  "description": "OIDC federation for Meridian Site pull requests",
  "audiences": ["api://AzureADTokenExchange"]
}'

# Least-privilege role assignment, scoped to the resource group
az role assignment create \
  --assignee <APP_ID> \
  --role Contributor \
  --scope /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP>
```

### 2. Configure GitHub repository variables

Add these as repository **variables** (they are identifiers, not secrets):

| Variable | Description |
|---|---|
| `AZURE_CLIENT_ID` | App registration (client) ID |
| `AZURE_TENANT_ID` | Entra ID tenant ID |
| `AZURE_SUBSCRIPTION_ID` | Target subscription ID |

### 3. CI/CD execution

The workflow in `.github/workflows/deploy.yml` runs automatically:

- **Push to `main`:** production deployment
- **Pull request:** isolated preview environment, removed when the PR closes

The job needs `permissions: id-token: write` to request the OIDC token, and authenticates with `azure/login` using the IDs above (no client secret).

---

## 📈 Impact & Business Value

- **No long-lived secrets:** No static Azure credentials are stored in GitHub, which removes the most common credential-leak path in the pipeline.
- **Security alignment:** Follows Microsoft Azure Security Benchmark guidance and zero-trust identity principles.
- **Lower maintenance:** Eliminates manual 90-day secret rotation cycles.

---

## 🔗 Part of a Larger Ecosystem

> ℹ️ This project is the **Azure Identity & Delivery** pattern within my broader multi-cloud portfolio (AWS, GCP, Azure).