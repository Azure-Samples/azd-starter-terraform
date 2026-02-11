# Copilot Instructions for AZD Terraform Starter

## Project Context

This is an Azure Developer CLI (azd) starter template using **Terraform** as the infrastructure provider. The template provisions Azure resources using AzureRM provider 4.x and integrates with the AZD workflow.

## Key Technologies

- **Azure Developer CLI (azd)**: Orchestrates provisioning and deployment
- **Terraform**: Infrastructure as Code (>= 1.1.7)
- **AzureRM Provider**: ~>4.21 (major version 4.x)
- **AzureCAF Provider**: ~>1.2.24 (resource naming conventions)

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

### Provider-Defined Functions (4.x)

New in 4.x - use these helper functions:
- `provider::azurerm::normalise_resource_id(id)` - Normalize resource IDs
- `provider::azurerm::parse_resource_id(id)` - Parse resource ID components

## Project Structure

```
├── azure.yaml              # AZD configuration (infra.provider: terraform)
├── infra/
│   ├── provider.tf         # Provider configuration (single source of truth)
│   ├── main.tf             # Main resources, locals, resource group
│   ├── variables.tf        # Input variables (location, environment_name)
│   ├── output.tf           # Outputs (saved to .env by azd)
│   └── core/               # Reusable child modules
│       ├── database/       # cosmos/, postgresql/
│       ├── gateway/        # apim/, apim-api/
│       ├── host/           # appserviceplan/, appservice/
│       ├── monitor/        # applicationinsights/, loganalytics/
│       └── security/       # keyvault/
```

## Required Tags for AZD Integration

Always include these tags on resources:

```hcl
tags = {
  azd-env-name = var.environment_name  # Required for azd
}
```

For service hosts (App Service, Function App), add:
```hcl
tags = {
  azd-env-name     = var.environment_name
  azd-service-name = "<service-name-from-azure.yaml>"
}
```

## Terraform Workflow

Always follow this sequence:

1. `terraform init` - Initialize providers
2. `terraform validate` - Validate configuration
3. `terraform plan` - Preview changes
4. `terraform apply -auto-approve` - Apply changes

Or use AZD commands:
```bash
azd provision  # Runs terraform init + apply
azd deploy     # Deploys application code
azd up         # Provision + Deploy
```

## Module Conventions

Each child module should:

1. Declare its own `required_providers` block
2. Use `azurecaf_name` for resource naming
3. Accept `resource_token`, `location`, `rg_name`, `tags` as standard inputs
4. Output resource IDs and connection strings (not secrets!)

Example module pattern:
```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~>4.21"
    }
    azurecaf = {
      source  = "aztfmod/azurecaf"
      version = "~>1.2.24"
    }
  }
}

resource "azurecaf_name" "resource_name" {
  name          = var.resource_token
  resource_type = "azurerm_<resource_type>"
  random_length = 0
  clean_input   = true
}
```

## Documentation References

- [AZD Overview](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/overview)
- [AZD Schema](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/azd-schema)
- [AzureRM Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [AzureRM 4.x Upgrade Guide](https://github.com/hashicorp/terraform-provider-azurerm/blob/main/website/docs/guides/4.0-upgrade-guide.html.markdown)
- [Terraform Style Guide](https://developer.hashicorp.com/terraform/language/style)
- [AzureRM Version History](https://learn.microsoft.com/en-us/azure/developer/terraform/provider-version-history-azurerm-4-0-0-to-current)

## Common Pitfalls to Avoid

1. **Don't use `skip_provider_registration`** - This is 3.x syntax. Use `resource_provider_registrations` instead.

2. **Don't hardcode subscription IDs** - Use `data.azurerm_client_config.current` for tenant/subscription info.

3. **Don't store secrets in outputs** - Outputs are saved to `.env` file. Use Key Vault references.

4. **Don't forget the `features {}` block** - It's required even if empty.

5. **Don't mix provider versions** - All modules should use the same `~>4.21` version constraint.

## Azure Authentication

Terraform uses Azure CLI credentials by default. Ensure the user has run:
```bash
az login
az account set --subscription "<SUBSCRIPTION_ID>"
```

## Local Development Setup

Required tools:
- Terraform >= 1.1.7 (`terraform version`)
- Azure CLI (`az version`)
- Azure Developer CLI (`azd version`)
