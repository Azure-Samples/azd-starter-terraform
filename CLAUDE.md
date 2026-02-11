# CLAUDE.md - Agent Instructions for AZD Terraform Starter

## Project Overview

This is an Azure Developer CLI (azd) starter template using **Terraform** with **AzureRM Provider 4.x**. It demonstrates best practices for provisioning Azure resources with modern Terraform patterns.

## Technology Stack

| Component | Version | Purpose |
|-----------|---------|---------|
| Terraform | >= 1.1.7 | IaC engine |
| AzureRM Provider | ~>4.21 | Azure resource management |
| AzureCAF Provider | ~>1.2.24 | Resource naming |
| Azure Developer CLI | latest | Orchestration |

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
   - Options: `"none"`, `"core"`, `"extended"`, `"all"`

2. **Provider Version Constraint**
   - ❌ OLD: `version = "~>3.97.1"`
   - ✅ NEW: `version = "~>4.21"`

3. **Source Order** (style preference)
   - ✅ Preferred: `source` before `version` in provider blocks

### Provider-Defined Functions (4.x Feature)

```hcl
# Normalize resource IDs
locals {
  normalized = provider::azurerm::normalise_resource_id(var.resource_id)
  parsed     = provider::azurerm::parse_resource_id(var.resource_id)
}
```

## Project File Structure

```
azd-starter-terraform/
├── azure.yaml              # AZD project config
├── infra/
│   ├── provider.tf         # Main provider (edit versions here)
│   ├── main.tf             # Resource group, locals
│   ├── variables.tf        # location, environment_name
│   ├── output.tf           # AZURE_LOCATION, AZURE_TENANT_ID
│   └── core/
│       ├── database/       # cosmos/, postgresql/
│       ├── gateway/        # apim/, apim-api/
│       ├── host/           # appserviceplan/, appservice/{node,java,python}/
│       ├── monitor/        # applicationinsights/, loganalytics/
│       └── security/       # keyvault/
```

## Modification Guidelines

### Adding a New Resource

1. Choose appropriate module directory (or create new)
2. Create `<resource>.tf`, `<resource>_variables.tf`, `<resource>_output.tf`
3. Include provider block with version constraint:
   ```hcl
   terraform {
     required_providers {
       azurerm = {
         source  = "hashicorp/azurerm"
         version = "~>4.21"
       }
     }
   }
   ```
4. Use `azurecaf_name` for naming
5. Include `tags` with `azd-env-name`

### Updating Provider Versions

When updating the AzureRM version, update ALL files:
- `infra/provider.tf`
- All `infra/core/**/*.tf` files with `required_providers` blocks

Use grep to find all occurrences:
```bash
grep -r "~>4.21" infra/
```

## Validation Workflow

Always validate after changes:

```bash
cd infra
terraform init -upgrade
terraform validate
terraform plan
```

## AZD Integration

The `azure.yaml` specifies Terraform as provider:
```yaml
infra:
  provider: terraform
```

AZD automatically:
- Runs `terraform init` and `terraform apply`
- Saves outputs to `.env` file
- Handles state management

## Common Tasks

### Initialize Environment
```bash
azd init
azd auth login
```

### Provision Infrastructure
```bash
azd provision
```

### Full Deployment
```bash
azd up
```

### Destroy Resources
```bash
azd down
```

## Documentation Links

- [AzureRM 4.0 Upgrade Guide](https://github.com/hashicorp/terraform-provider-azurerm/blob/main/website/docs/guides/4.0-upgrade-guide.html.markdown)
- [AzureRM Provider Docs](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [AzureRM Version History](https://learn.microsoft.com/en-us/azure/developer/terraform/provider-version-history-azurerm-4-0-0-to-current)
- [AZD Documentation](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/overview)
- [Terraform Style Guide](https://developer.hashicorp.com/terraform/language/style)

## Anti-Patterns to Avoid

1. ❌ Using `skip_provider_registration` (3.x syntax)
2. ❌ Hardcoding subscription/tenant IDs
3. ❌ Outputting secrets (use Key Vault)
4. ❌ Missing `features {}` block
5. ❌ Inconsistent provider versions across modules
6. ❌ Missing `azd-env-name` tag on resources
