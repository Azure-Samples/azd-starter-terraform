# Copilot Instructions for AZD Terraform Starter

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
- **Agent instruction files** — `.github/copilot-instructions.md` (this file) and `CLAUDE.md`
- **`.devcontainer/devcontainer.json`** — adds Terraform feature/extension compared to source

### When propagating updates from the source template:
1. Pull latest changes from [functions-quickstart-dotnet-azd](https://github.com/Azure-Samples/functions-quickstart-dotnet-azd)
2. Copy over the "stays in sync" files listed above
3. Adjust `.vscode/settings.json` and `.vscode/tasks.json` paths for the .NET version if changed
4. Update `infra/` Terraform to match any new Azure resources or configuration changes from the Bicep version
5. Update `README.md` sections that mirror the source (usage, source code, etc.)
6. Do NOT overwrite agent instruction files or Terraform-specific config

## Project Context

This is an Azure Developer CLI (azd) starter template using **Terraform** as the infrastructure provider. The template provisions a **Flex Consumption Function App** with supporting Azure resources using AzureRM provider 4.x.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ Resource Group (rg-{environment_name})                         │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐  ┌──────────────────┐                    │
│  │ User Assigned    │  │ Storage Account  │                    │
│  │ Managed Identity │  │ (deployment pkg) │                    │
│  └────────┬─────────┘  └────────┬─────────┘                    │
│           │                     │                               │
│           ▼                     ▼                               │
│  ┌─────────────────────────────────────────────────────┐       │
│  │ Function App (Flex Consumption - FC1)               │       │
│  │ - dotnet-isolated 10.0                              │       │
│  │ - HTTP Triggers (httpget, httppost)                  │       │
│  └─────────────────────────────────────────────────────┘       │
│           │                                                     │
│           ▼                                                     │
│  ┌──────────────────┐  ┌──────────────────┐                    │
│  │ Log Analytics    │◄─│ Application      │                    │
│  │ Workspace        │  │ Insights (AAD)   │                    │
│  └──────────────────┘  └──────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

## Key Technologies

- **Azure Developer CLI (azd)**: Orchestrates provisioning and deployment
- **Terraform**: Infrastructure as Code (>= 1.1.7)
- **AzureRM Provider**: ~>4.21 (major version 4.x)
- **AzureCAF Provider**: ~>1.2.24 (resource naming conventions)

## Flex Consumption Function App (NEW in 4.x)

### Resource: `azurerm_function_app_flex_consumption`

This is the key resource for Flex Consumption Function Apps. Example pattern:

```hcl
resource "azurerm_function_app_flex_consumption" "api" {
  name                = "func-api-${local.resource_token}"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name
  service_plan_id     = azurerm_service_plan.flex_plan.id

  # Storage for deployment package (REQUIRED)
  storage_container_type            = "blobContainer"
  storage_container_endpoint        = "${azurerm_storage_account.storage.primary_blob_endpoint}${azurerm_storage_container.deployment.name}"
  storage_authentication_type       = "UserAssignedIdentity"
  storage_user_assigned_identity_id = azurerm_user_assigned_identity.api.id

  # Runtime
  runtime_name    = "dotnet-isolated"
  runtime_version = "10.0"

  # Scaling
  maximum_instance_count = 100
  instance_memory_in_mb  = 2048

  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.api.id]
  }

  site_config {}

  app_settings = {
    "AzureWebJobsStorage__credential"     = "managedidentity"
    "AzureWebJobsStorage__clientId"       = azurerm_user_assigned_identity.api.client_id
    "AzureWebJobsStorage__blobServiceUri" = azurerm_storage_account.storage.primary_blob_endpoint
  }

  tags = merge(local.tags, { "azd-service-name" = "api" })
}
```

### Service Plan for Flex Consumption

```hcl
resource "azurerm_service_plan" "flex_plan" {
  name                = "plan-${local.resource_token}"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name
  os_type             = "Linux"
  sku_name            = "FC1"  # Flex Consumption SKU
  tags                = local.tags
}
```

## Critical Rules for Terraform 4.x

### Provider Configuration

Always use this provider block pattern for AzureRM 4.x:

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
  resource_provider_registrations = "none"  # or "core", "extended", "all"
  features {
    key_vault {
      purge_soft_delete_on_destroy = false
    }
    resource_group {
      prevent_deletion_if_contains_resources = false
    }
  }
}
```

### 3.x to 4.x Migration Patterns

| 3.x Pattern | 4.x Pattern |
|-------------|-------------|
| `skip_provider_registration = "true"` | `resource_provider_registrations = "none"` |
| `version = "~>3.97.1"` | `version = "~>4.21"` |
| `azurerm_function_app` | `azurerm_function_app_flex_consumption` (for Flex) |

## RBAC Role Assignments

For managed identity authentication to work, assign these roles:

```hcl
# Storage Blob Data Owner - for deployment packages
resource "azurerm_role_assignment" "storage_blob_data_owner" {
  scope                = azurerm_storage_account.storage.id
  role_definition_name = "Storage Blob Data Owner"
  principal_id         = azurerm_user_assigned_identity.api.principal_id
  principal_type       = "ServicePrincipal"
}

# Monitoring Metrics Publisher - for App Insights
resource "azurerm_role_assignment" "monitoring_metrics_publisher" {
  scope                = azurerm_application_insights.appinsights.id
  role_definition_name = "Monitoring Metrics Publisher"
  principal_id         = azurerm_user_assigned_identity.api.principal_id
  principal_type       = "ServicePrincipal"
}
```

## Required Tags for AZD Integration

Always include these tags on resources:

```hcl
tags = {
  azd-env-name = var.environment_name  # Required for azd
}
```

For service hosts (Function App), add:
```hcl
tags = {
  azd-env-name     = var.environment_name
  azd-service-name = "api"  # Must match service name in azure.yaml
}
```

## Documentation References

- [azurerm_function_app_flex_consumption](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/function_app_flex_consumption)
- [Azure Functions Flex Consumption](https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan)
- [AzureRM 4.x Upgrade Guide](https://github.com/hashicorp/terraform-provider-azurerm/blob/main/website/docs/guides/4.0-upgrade-guide.html.markdown)
- [Terraform AzureRM Provider Registry](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [AZD Overview](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/overview)

## Security Best Practices - Disable Local Auth

**CRITICAL**: Always disable local/key-based authentication for Azure resources and use managed identity instead.

### Storage Account

```hcl
resource "azurerm_storage_account" "storage" {
  name                            = "st${local.resource_token}"
  resource_group_name             = azurerm_resource_group.rg.name
  location                        = var.location
  account_tier                    = "Standard"
  account_replication_type        = "LRS"
  
  # Security: Disable local authentication
  shared_access_key_enabled       = false  # REQUIRED - Disable key-based auth
  default_to_oauth_authentication = true   # Use Entra ID by default
  allow_nested_items_to_be_public = false  # No public blob access
  min_tls_version                 = "TLS1_2"
}
```

### Application Insights

```hcl
resource "azurerm_application_insights" "appinsights" {
  name                = "appi-${local.resource_token}"
  location            = var.location
  resource_group_name = azurerm_resource_group.rg.name
  workspace_id        = azurerm_log_analytics_workspace.workspace.id
  application_type    = "web"
  
  # Security: Disable local authentication
  local_authentication_disabled = true  # Use Entra ID/RBAC only
}
```

### Provider Configuration for Entra ID Storage Access

When using `shared_access_key_enabled = false`, configure the provider to use Entra ID for storage data plane operations:

```hcl
provider "azurerm" {
  resource_provider_registrations = "none"
  storage_use_azuread             = true  # Use Entra ID for storage operations
  features {
    # ...
  }
}
```

### RBAC for Deploying User

When disabling shared keys, you must grant the deploying user/identity Storage Blob Data Owner BEFORE creating containers:

```hcl
resource "azurerm_role_assignment" "storage_blob_data_owner_deployer" {
  scope                = azurerm_storage_account.storage.id
  role_definition_name = "Storage Blob Data Owner"
  principal_id         = data.azurerm_client_config.current.object_id
  principal_type       = "User"
}

resource "azurerm_storage_container" "deployment_package" {
  name               = "app-package"
  storage_account_id = azurerm_storage_account.storage.id
  
  depends_on = [azurerm_role_assignment.storage_blob_data_owner_deployer]
}
```

### Why This Matters

- Azure Policy may **block** resources with local auth enabled (403 RequestDisallowedByPolicy)
- Managed identity is more secure - no credentials to rotate or leak
- This is Azure's recommended security posture for production workloads

## Common Pitfalls to Avoid

1. **Don't use `skip_provider_registration`** - This is 3.x syntax. Use `resource_provider_registrations` instead.

2. **Don't forget RBAC assignments** - Flex Consumption requires Storage Blob Data Owner role for deployment.

3. **Don't use `shared_access_key_enabled = true`** - Use managed identity for security.

4. **Don't forget `azd-service-name` tag** - Required for AZD to deploy to the correct resource.

5. **Don't mix provider versions** - All modules should use the same `~>4.21` version constraint.

6. **Don't forget `storage_use_azuread = true`** - Required in provider when storage has local auth disabled.

7. **Don't create storage containers before RBAC** - Add `depends_on` for deployer role assignment.

## Azure Authentication

Terraform uses Azure CLI credentials by default. Ensure the user has run:
```bash
az login
az account set --subscription "<SUBSCRIPTION_ID>"
```

