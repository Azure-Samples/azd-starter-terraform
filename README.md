# Azure Developer CLI (azd) Terraform Starter - Flex Consumption Function

A starter blueprint for deploying **Azure Functions with Flex Consumption** using [Azure Developer CLI](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/overview) (azd) and **Terraform**. This template demonstrates modern infrastructure patterns with AzureRM Provider 4.x.

**Migrated from**: [functions-quickstart-dotnet-azd](https://github.com/Azure-Samples/functions-quickstart-dotnet-azd) (Bicep → Terraform)

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ Resource Group (rg-{environment})                               │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐  ┌──────────────────┐                    │
│  │ User Assigned    │  │ Storage Account  │                    │
│  │ Managed Identity │  │ (deployment pkg) │                    │
│  └────────┬─────────┘  └────────┬─────────┘                    │
│           │                     │                               │
│           ▼                     ▼                               │
│  ┌─────────────────────────────────────────────────────┐       │
│  │ Function App (Flex Consumption - FC1 SKU)           │       │
│  │ - .NET 10 isolated worker                           │       │
│  │ - HTTP Trigger (httpget)                            │       │
│  └─────────────────────────────────────────────────────┘       │
│           │                                                     │
│           ▼                                                     │
│  ┌──────────────────┐  ┌──────────────────┐                    │
│  │ Log Analytics    │◄─│ Application      │                    │
│  │ Workspace        │  │ Insights (AAD)   │                    │
│  └──────────────────┘  └──────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

## What's Deployed

| Resource | Description |
|----------|-------------|
| Resource Group | Container for all resources |
| User Assigned Managed Identity | Passwordless authentication |
| Storage Account | Blob container for function deployment packages |
| Log Analytics Workspace | Centralized logging |
| Application Insights | APM with local auth disabled |
| App Service Plan (FC1) | Flex Consumption hosting plan |
| Function App | .NET 10 isolated HTTP trigger function |

## Prerequisites

### Install Required Tools

1. **Terraform** (>= 1.1.7)
   ```bash
   # macOS
   brew install terraform
   
   # Windows
   winget install Hashicorp.Terraform
   
   # Linux (Ubuntu/Debian)
   sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
   wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
   echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
   sudo apt update && sudo apt install terraform
   ```

2. **.NET 10 SDK** (for building the function)
   ```bash
   # macOS
   brew install dotnet@10
   
   # Windows
   winget install Microsoft.DotNet.SDK.10
   ```

3. **Azure CLI**
   ```bash
   # macOS
   brew install azure-cli
   
   # Windows
   winget install Microsoft.AzureCLI
   
   # Linux
   curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
   ```

4. **Azure Developer CLI (azd)**
   ```bash
   # macOS
   brew install azure-dev
   
   # Windows
   winget install Microsoft.Azd
   
   # Linux
   curl -fsSL https://aka.ms/install-azd.sh | bash
   ```

### Configure Azure Authentication

```bash
# Login to Azure CLI (required for Terraform auth)
az login

# Set your subscription
az account set --subscription "<YOUR_SUBSCRIPTION_ID>"

# Verify current subscription
az account show --query "{name:name, id:id, tenantId:tenantId}"
```

## Provider Configuration

This template uses **AzureRM Provider 4.x** with the following key features:

- **Terraform**: >= 1.1.7
- **AzureRM Provider**: ~>4.21
- **AzureCAF Provider**: ~>1.2.24 (for resource naming)

### Key 4.x Changes from 3.x

| Setting (3.x) | Setting (4.x) |
|---------------|---------------|
| `skip_provider_registration = true` | `resource_provider_registrations = "none"` |

For full migration guide, see: [AzureRM 4.0 Upgrade Guide](https://github.com/hashicorp/terraform-provider-azurerm/blob/main/website/docs/guides/4.0-upgrade-guide.html.markdown)

## Documentation References

- [Azure Functions Flex Consumption](https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan)
- [azurerm_function_app_flex_consumption](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/function_app_flex_consumption)
- [Azure Developer CLI Overview](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/overview)
- [Terraform AzureRM Provider Docs](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [AzureRM 4.x Upgrade Guide](https://github.com/hashicorp/terraform-provider-azurerm/blob/main/website/docs/guides/4.0-upgrade-guide.html.markdown)

## Project Structure

```
├── azure.yaml              # AZD project configuration
├── http/                   # Function app source code
│   ├── http.csproj         # .NET 10 project file
│   ├── Program.cs          # Host configuration
│   ├── httpGetFunction.cs  # HTTP trigger function
│   └── host.json           # Function host settings
├── infra/
│   ├── provider.tf         # Provider configuration (azurerm ~>4.21)
│   ├── main.tf             # All infrastructure resources
│   ├── variables.tf        # Input variables
│   └── output.tf           # Output values (for AZD)
└── .devcontainer/          # Dev container with tools pre-installed
```

The following assets have been provided:

- **Function App Code** in `http/` - A .NET 10 isolated worker HTTP trigger function
- **Infrastructure-as-code** in `infra/` - Terraform configuration for Flex Consumption Function App
- **Dev Container** in `.devcontainer/` - Pre-configured development environment

## Quick Start

```bash
# Login to Azure
az login
az account set --subscription "<YOUR_SUBSCRIPTION_ID>"

# Deploy everything with AZD
azd up

# Or step-by-step:
azd provision  # Create infrastructure
azd deploy     # Deploy function code
```

## Test the Function

After deployment, test your function:

```bash
# Get the function URL
azd env get-values | grep AZURE_FUNCTION_NAME

# Test the HTTP endpoint
curl "https://<function-name>.azurewebsites.net/api/httpget?name=World"
```

## Key Features

### Flex Consumption Plan
- **Automatic scaling** from 0 to 100 instances
- **VNet integration** support (optional)
- **Instance memory**: 2048 MB
- **Pay-per-use** billing

### Managed Identity Authentication
- User Assigned Managed Identity for all Azure access
- No connection strings or access keys
- RBAC roles: Storage Blob Data Owner, Monitoring Metrics Publisher

### Application Insights
- Local authentication disabled (AAD only)
- Connected to Log Analytics workspace
- Automatic telemetry collection

## Customization

### Change Runtime

Edit `infra/main.tf`:
```hcl
runtime_name    = "node"      # or "python", "java", "dotnet-isolated"
runtime_version = "20"        # version appropriate for runtime
```

### Adjust Scaling

```hcl
maximum_instance_count = 100  # Max instances
instance_memory_in_mb  = 2048 # Memory per instance
```

### Enable VNet Integration

VNet integration is not enabled by default. To add it, you would need to:
1. Create a Virtual Network and subnet
2. Add `virtual_network_subnet_id` to the function app resource

## Additional Details

### Managed Identities

This template uses a **User Assigned Managed Identity** for passwordless authentication to Azure services:

- **Storage Account**: Storage Blob Data Owner role for deployment packages
- **Application Insights**: Monitoring Metrics Publisher role for telemetry

### Azure Key Vault

[Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/general/overview) allows you to store secrets securely. Your application can access these secrets securely through the use of managed identities.

### Flex Consumption vs. Consumption Plan

| Feature | Flex Consumption | Classic Consumption |
|---------|------------------|---------------------|
| VNet Integration | ✅ Yes | ❌ No |
| Custom Scaling | ✅ Yes (max instances) | ❌ No |
| Instance Memory | ✅ Configurable | ❌ Fixed |
| Cold Start | ✅ Reduced | ⚠️ Higher |

## Cleanup

```bash
# Remove all deployed resources
azd down
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.
