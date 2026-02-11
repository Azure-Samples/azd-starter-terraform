# CLAUDE.md - Agent Instructions for AZD Terraform Starter

## Project Overview

This is an Azure Developer CLI (azd) starter template using **Terraform** with **AzureRM Provider 4.x**. It deploys a **Flex Consumption Function App** - a modern serverless compute option with enhanced scaling and VNet support.

**Migrated from**: https://github.com/Azure-Samples/functions-quickstart-dotnet-azd/tree/main/infra

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
  runtime_version = "8.0"

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
├── azure.yaml              # AZD project config (services: api)
├── http/                   # Function app code
│   ├── http.csproj         # .NET 10 isolated worker
│   ├── Program.cs          # Host configuration
│   ├── httpGetFunction.cs  # HTTP trigger function
│   └── host.json           # Function host settings
├── infra/
│   ├── provider.tf         # Provider configuration (~>4.21)
│   ├── main.tf             # All resources in single file
│   ├── variables.tf        # location, environment_name, principal_id
│   └── output.tf           # AZD outputs (SERVICE_API_NAME, etc.)
└── .github/
    └── copilot-instructions.md
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

## Anti-Patterns to Avoid

1. ❌ Using `skip_provider_registration` (3.x syntax)
2. ❌ Using `shared_access_key_enabled = true` on storage
3. ❌ Missing `azd-service-name` tag on function app
4. ❌ Forgetting RBAC role assignments before creating function
5. ❌ Using `azurerm_function_app` for Flex Consumption
6. ❌ Missing `depends_on` for role assignments
