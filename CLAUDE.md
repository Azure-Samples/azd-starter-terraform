# CLAUDE.md - Agent Instructions for AZD Terraform Starter

## Provenance & Maintenance

This template is a **Terraform port** of the Bicep-based [functions-quickstart-dotnet-azd](https://github.com/Azure-Samples/functions-quickstart-dotnet-azd).

### What stays in sync with the source Bicep template:
- **Function app source code** (`http/` folder) — should match the source repo exactly (except .NET version choice)
- **Root config files** — `.gitignore`, `CONTRIBUTING.md`, `LICENSE.md`, `CHANGELOG.md`, `.github/CODE_OF_CONDUCT.md`, `.github/ISSUE_TEMPLATE.md`, `.github/PULL_REQUEST_TEMPLATE.md`
- **VS Code configuration** (`.vscode/` folder) — extensions, launch, tasks, settings
- **Dev container** (`.devcontainer/` folder) — same as source plus Terraform tooling
- **Solution file** (`http.sln`)

### What is Terraform-specific (differs from source):
- **`infra/` folder** — Terraform (HCL) instead of Bicep; uses AzureRM Provider 4.x and AzureCAF provider
- **`azure.yaml`** — includes `infra: provider: terraform`
- **`README.md`** — adapted from source with Terraform-specific sections and prerequisites
- **Agent instruction files** — `CLAUDE.md` (this file) and `.github/copilot-instructions.md`
- **`.devcontainer/devcontainer.json`** — adds Terraform feature/extension compared to source

### When propagating updates from the source template:
1. Pull latest changes from [functions-quickstart-dotnet-azd](https://github.com/Azure-Samples/functions-quickstart-dotnet-azd)
2. Copy over the "stays in sync" files listed above
3. Adjust `.vscode/settings.json` and `.vscode/tasks.json` paths for the .NET version if changed
4. Update `infra/` Terraform to match any new Azure resources or configuration changes from the Bicep version
5. Update `README.md` sections that mirror the source (usage, source code, etc.)
6. Do NOT overwrite agent instruction files or Terraform-specific config

## Project Overview

This is an Azure Developer CLI (azd) starter template using **Terraform** with **AzureRM Provider 4.x**. It deploys a **Flex Consumption Function App** - a modern serverless compute option with enhanced scaling and VNet support.

## Architecture

| Resource | Terraform Resource | Purpose |
|----------|-------------------|---------|
| Resource Group | `azurerm_resource_group` | Container for all resources |
| User Assigned Identity | `azurerm_user_assigned_identity` | Passwordless auth |
| Storage Account | `azurerm_storage_account` | Function deployment packages |
| Log Analytics | `azurerm_log_analytics_workspace` | Centralized logging |
| App Insights | `azurerm_application_insights` | APM (local auth disabled) |
| Service Plan | `azurerm_service_plan` (SKU: FC1) | Flex Consumption plan |
| Function App | `azurerm_function_app_flex_consumption` | HTTP-triggered function |

## Technology Stack

| Component | Version | Purpose |
|-----------|---------|---------|
| Terraform | >= 1.1.7 | IaC engine |
| AzureRM Provider | ~>4.21 | Azure resource management |
| AzureCAF Provider | ~>1.2.24 | Resource naming |
| Azure Developer CLI | latest | Orchestration |

## Key Resource: Flex Consumption Function App

### `azurerm_function_app_flex_consumption`

This is NEW in AzureRM 4.21+. Key attributes:

```hcl
resource "azurerm_function_app_flex_consumption" "api" {
  name                = "func-api-${local.resource_token}"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name
  service_plan_id     = azurerm_service_plan.flex_plan.id

  # REQUIRED: Storage for deployment packages
  storage_container_type            = "blobContainer"
  storage_container_endpoint        = "${storage.primary_blob_endpoint}${container.name}"
  storage_authentication_type       = "UserAssignedIdentity"
  storage_user_assigned_identity_id = azurerm_user_assigned_identity.api.id

  # Runtime
  runtime_name    = "dotnet-isolated"  # or "node", "python", "java"
  runtime_version = "10.0"

  # Scaling (Flex Consumption specific)
  maximum_instance_count = 100
  instance_memory_in_mb  = 2048

  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.api.id]
  }

  site_config {}  # Required even if empty

  app_settings = {
    "AzureWebJobsStorage__credential"     = "managedidentity"
    "AzureWebJobsStorage__clientId"       = identity.client_id
    "AzureWebJobsStorage__blobServiceUri" = storage.primary_blob_endpoint
  }

  tags = { "azd-service-name" = "api" }
}
```

### Service Plan for Flex Consumption

```hcl
resource "azurerm_service_plan" "flex_plan" {
  name                = "plan-${local.resource_token}"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name
  os_type             = "Linux"
  sku_name            = "FC1"  # IMPORTANT: Flex Consumption SKU
  tags                = local.tags
}
```

## Critical: AzureRM 4.x Patterns

### Provider Block (REQUIRED)

```hcl
terraform {
  required_version = ">= 1.1.7, < 2.0.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>4.21"
    }
  }
}

provider "azurerm" {
  resource_provider_registrations = "none"
  features {}
}
```

### Key Breaking Changes from 3.x

1. **Resource Provider Registration** (CRITICAL)
   - ❌ OLD (3.x): `skip_provider_registration = "true"`
   - ✅ NEW (4.x): `resource_provider_registrations = "none"`

2. **Flex Consumption Function App**
   - ❌ OLD: `azurerm_function_app` (doesn't support Flex)
   - ✅ NEW: `azurerm_function_app_flex_consumption`

## RBAC Requirements

For managed identity auth to work, these roles are **REQUIRED**:

| Role | Scope | Purpose |
|------|-------|---------|
| Storage Blob Data Owner | Storage Account | Deployment packages |
| Monitoring Metrics Publisher | App Insights | Telemetry publishing |

```hcl
resource "azurerm_role_assignment" "storage_blob_data_owner" {
  scope                = azurerm_storage_account.storage.id
  role_definition_name = "Storage Blob Data Owner"
  principal_id         = azurerm_user_assigned_identity.api.principal_id
  principal_type       = "ServicePrincipal"
}
```

## Project File Structure

```
azd-starter-terraform/
├── azure.yaml              # AZD project config (services: api) + infra: provider: terraform
├── http.sln                # Visual Studio solution file
├── http/                   # Function app code (synced from source Bicep template)
│   ├── http.csproj         # .NET 10 isolated worker
│   ├── Program.cs          # Host configuration
│   ├── httpGetFunction.cs  # HTTP GET trigger function
│   ├── httpPostBodyFunction.cs # HTTP POST trigger function
│   ├── host.json           # Function host settings
│   ├── test.http           # HTTP test file for VS Code REST Client
│   ├── testdata.json       # Sample POST payload
│   └── Properties/         # VS launch/service dependency settings
├── infra/                  # Terraform IaC (this is what differs from source)
│   ├── provider.tf         # Provider configuration (~>4.21)
│   ├── main.tf             # All resources in single file
│   ├── variables.tf        # location, environment_name, principal_id
│   ├── output.tf           # AZD outputs (SERVICE_API_NAME, etc.)
│   └── main.tfvars.json    # Default variable values
├── .devcontainer/          # Dev container (source + Terraform tooling)
├── .vscode/                # VS Code config (synced from source)
├── .github/
│   ├── copilot-instructions.md  # Copilot agent instructions (Terraform-specific)
│   ├── CODE_OF_CONDUCT.md       # Synced from source
│   ├── ISSUE_TEMPLATE.md        # Synced from source
│   └── PULL_REQUEST_TEMPLATE.md # Synced from source
├── CLAUDE.md               # Claude agent instructions (this file, Terraform-specific)
├── CONTRIBUTING.md         # Synced from source
├── LICENSE.md              # Synced from source
├── CHANGELOG.md            # Synced from source
└── .gitignore              # Source .gitignore + Terraform patterns
```

## AZD Integration

The `azure.yaml` configures the function service:
```yaml
name: azd-starter-terraform
services:
  api:
    project: ./http/
    language: dotnet
    host: function
infra:
  provider: terraform
```

**Critical Tag**: Function App must have `azd-service-name = "api"` tag to match the service name.

## Common Tasks

### Deploy Everything
```bash
azd up
```

### Just Infrastructure
```bash
azd provision
```

### Just Application Code
```bash
azd deploy
```

### Destroy
```bash
azd down
```

## Documentation Links

- [azurerm_function_app_flex_consumption](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/function_app_flex_consumption)
- [Azure Functions Flex Consumption](https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan)
- [AzureRM 4.0 Upgrade Guide](https://github.com/hashicorp/terraform-provider-azurerm/blob/main/website/docs/guides/4.0-upgrade-guide.html.markdown)
- [AZD Documentation](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/overview)

## Security Best Practices - Disable Local Auth

**CRITICAL**: Always disable local/key-based authentication. Azure Policy may block resources with local auth enabled.

### Storage Account Configuration

```hcl
resource "azurerm_storage_account" "storage" {
  name                            = "st${local.resource_token}"
  # ...
  shared_access_key_enabled       = false  # REQUIRED - Disable key-based auth
  default_to_oauth_authentication = true   # Use Entra ID by default
  allow_nested_items_to_be_public = false  # No public blob access
}
```

### Application Insights Configuration

```hcl
resource "azurerm_application_insights" "appinsights" {
  name                          = "appi-${local.resource_token}"
  # ...
  local_authentication_disabled = true  # Use Entra ID/RBAC only
}
```

### Provider Configuration for Keyless Storage

```hcl
provider "azurerm" {
  resource_provider_registrations = "none"
  storage_use_azuread             = true  # REQUIRED when shared keys disabled
  features {}
}
```

### Deployer RBAC (Order Matters!)

When using `shared_access_key_enabled = false`, assign RBAC BEFORE creating containers:

```hcl
resource "azurerm_role_assignment" "storage_blob_data_owner_deployer" {
  scope                = azurerm_storage_account.storage.id
  role_definition_name = "Storage Blob Data Owner"
  principal_id         = data.azurerm_client_config.current.object_id
  principal_type       = "User"
}

resource "azurerm_storage_container" "deployment_package" {
  # ...
  depends_on = [azurerm_role_assignment.storage_blob_data_owner_deployer]  # CRITICAL
}
```

## Anti-Patterns to Avoid

1. ❌ Using `skip_provider_registration` (3.x syntax)
2. ❌ Using `shared_access_key_enabled = true` on storage
3. ❌ Missing `azd-service-name` tag on function app
4. ❌ Forgetting RBAC role assignments before creating function
5. ❌ Using `azurerm_function_app` for Flex Consumption
6. ❌ Missing `depends_on` for role assignments
7. ❌ Forgetting `storage_use_azuread = true` when storage has local auth disabled
8. ❌ Creating storage containers before deployer has RBAC access
