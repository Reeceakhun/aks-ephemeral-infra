# Azure OIDC Setup for GitHub Actions

This repo authenticates to Azure using OIDC federation — no client secret or
service principal password is stored anywhere. GitHub issues a short-lived
token per workflow run; Azure AD trusts it based on the federated credential
configured below.

## 1. Create an App Registration

```bash
az ad app create --display-name "aks-ephemeral-infra-github"
```

Note the returned `appId` — this is your `AZURE_CLIENT_ID`.

## 2. Create a Service Principal for the app

```bash
az ad sp create --id <appId>
```

## 3. Assign it a scoped role

Grant Contributor **only on the resource groups this repo will create/delete**,
not the whole subscription — least privilege:

```bash
az role assignment create \
  --assignee <appId> \
  --role Contributor \
  --scope /subscriptions/<subscription-id>
```
(Subscription-level scope is used here because resource groups are created
dynamically per environment and don't exist yet to scope against individually.
If you want tighter scoping, use a subscription-level custom role restricted
to resource-group create/delete + AKS operations instead of full Contributor.)

## 4. Add a federated credential trusting this GitHub repo

```bash
az ad app federated-credential create \
  --id <appId> \
  --parameters '{
    "name": "github-actions-main",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:Reeceakhun/aks-ephemeral-infra:ref:refs/heads/master",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

The `subject` field is what ties this specifically to this repo and branch —
no other repo can use this credential to authenticate.

## 5. Add repo secrets in GitHub

Settings → Secrets and variables → Actions:

| Secret | Value |
|---|---|
| `AZURE_CLIENT_ID` | the `appId` from step 1 |
| `AZURE_TENANT_ID` | output of `az account show --query tenantId -o tsv` |
| `AZURE_SUBSCRIPTION_ID` | output of `az account show --query id -o tsv` |

No client secret is stored — OIDC handles the trust at request time.

## What workflows need to use this

Every job that talks to Azure needs:
```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: azure/login@v2
    with:
      client-id: ${{ secrets.AZURE_CLIENT_ID }}
      tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```
