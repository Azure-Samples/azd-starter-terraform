# Azure Developer CLI (azd) Terraform Starter

A starter blueprint for getting your application up on Azure using [Azure Developer CLI](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/overview) (azd). Add your application code, write Infrastructure as Code assets in Terraform to get your application up and running quickly.

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

2. **Azure CLI**
   ```bash
   # macOS
   brew install azure-cli
   
   # Windows
   winget install Microsoft.AzureCLI
   
   # Linux
   curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
   ```

3. **Azure Developer CLI (azd)**
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

- [Azure Developer CLI Overview](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/overview)
- [AZD Schema Reference](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/azd-schema)
- [Terraform AzureRM Provider Docs](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [AzureRM 4.x Version History](https://learn.microsoft.com/en-us/azure/developer/terraform/provider-version-history-azurerm-4-0-0-to-current)
- [Terraform Style Guide](https://developer.hashicorp.com/terraform/language/style)

## Project Structure

```
├── azure.yaml              # AZD project configuration
├── infra/
│   ├── provider.tf         # Provider configuration (azurerm ~>4.21)
│   ├── main.tf             # Main infrastructure
│   ├── variables.tf        # Input variables
│   ├── output.tf           # Output values
│   └── core/               # Reusable modules
│       ├── database/       # CosmosDB, PostgreSQL
│       ├── gateway/        # API Management
│       ├── host/           # App Service, App Service Plans
│       ├── monitor/        # Application Insights, Log Analytics
│       └── security/       # Key Vault
└── .devcontainer/          # Dev container with tools pre-installed
```

The following assets have been provided:

- Infrastructure-as-code (IaC) Terraform modules under the `infra` directory that demonstrate how to provision resources and setup resource tagging for azd.
- A [dev container](https://containers.dev) configuration file under the `.devcontainer` directory that installs infrastructure tooling by default. This can be readily used to create cloud-hosted developer environments such as [GitHub Codespaces](https://aka.ms/codespaces).
- Continuous deployment workflows for CI providers such as GitHub Actions under the `.github` directory, and Azure Pipelines under the `.azdo` directory that work for most use-cases.

## Quick Start

```bash
# Initialize azd environment
azd init

# Validate Terraform configuration
cd infra && terraform init && terraform validate && cd ..

# Provision Azure resources
azd provision

# Deploy application (if services configured)
azd deploy

# Or do both at once
azd up
```

## Next Steps

### Step 1: Add application code

1. Initialize the service source code projects anywhere under the current directory. Ensure that all source code projects can be built successfully.
    - > Note: For `function` services, it is recommended to initialize the project using the provided [quickstart tools](https://learn.microsoft.com/en-us/azure/azure-functions/functions-get-started).
2. Once all service source code projects are building correctly, update `azure.yaml` to reference the source code projects.
3. Run `azd package` to validate that all service source code projects can be built and packaged locally.

### Step 2: Provision Azure resources

Update or add Terraform modules to provision the relevant Azure resources. This can be done incrementally, as the list of [Azure resources](https://learn.microsoft.com/en-us/azure/?product=popular) are explored and added.

- All Azure resources available in Terraform format can be found [here](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs).

Run `azd provision` whenever you want to ensure that changes made are applied correctly and work as expected.

### Step 3: Tie in application and infrastructure

Certain changes to Terraform modules or deployment manifests are required to tie in application and infrastructure together. For example:

1. Set up [application settings](#application-settings) for the code running in Azure to connect to other Azure resources.
1. If you are accessing sensitive resources in Azure, set up [managed identities](#managed-identities) to allow the code running in Azure to securely access the resources.
1. If you have secrets, it is recommended to store secrets in [Azure Key Vault](#azure-key-vault) that then can be retrieved by your application, with the use of managed identities.
1. Configure [host configuration](#host-configuration) on your hosting platform to match your application's needs. This may include networking options, security options, or more advanced configuration that helps you take full advantage of Azure capabilities.

For more details, see [additional details](#additional-details) below.

When changes are made, use azd to apply your changes in Azure and validate that they are working as expected:

- Run `azd up` to validate both infrastructure and application code changes.
- Run `azd deploy` to validate application code changes.

### Step 4: Up to Azure

Finally, run `azd up` to run the end-to-end infrastructure provisioning (`azd provision`) and deployment (`azd deploy`) flow. Visit the service endpoints listed to see your application up-and-running!

## Additional Details

The following section examines different concepts that help tie in application and infrastructure.

### Application settings

It is recommended to have application settings managed in Azure, separating configuration from code. Typically, the service host allows for application settings to be defined.

- For `appservice` and `function`, application settings should be defined on the Terraform resource for the targeted host. Reference template example [here](https://github.com/Azure-Samples/todo-nodejs-mongo-terraform/tree/main/infra).
- For `aks`, application settings are applied using deployment manifests under the `<service>/manifests` folder. Reference template example [here](https://github.com/Azure-Samples/todo-nodejs-mongo-aks/tree/main/src/api/manifests).

### Managed identities

[Managed identities](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview) allows you to secure communication between services. This is done without having the need for you to manage any credentials.

### Azure Key Vault

[Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/general/overview) allows you to store secrets securely. Your application can access these secrets securely through the use of managed identities.

### Host configuration

For `appservice`, the following host configuration options are often modified:

- Language runtime version
- Exposed port from the running container (if running a web service)
- Allowed origins for CORS (Cross-Origin Resource Sharing) protection (if running a web service backend with a frontend)
- The run command that starts up your service
