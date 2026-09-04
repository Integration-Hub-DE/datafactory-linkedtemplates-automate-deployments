# Data Factory Automatic Deployment using Linked ARM templates

### Overview

This project demonstrates a complete Continuous Integration and Continuous Deployment (CI/CD) implementation for Azure Data Factory (ADF) using GitHub Actions and Azure Resource Manager (ARM) templates.

The solution automates the deployment lifecycle of Azure Data Factory resources using Linked ARM Templates by validating ADF artifacts, generating ARM templates, storing deployment artifacts, and deploying changes to a target Data Factory environment whenever changes are merged into the repository's master branch.

The objective is to eliminate manual deployment activities and provide a repeatable, version-controlled, and auditable deployment process across environments.

### What The Workflow Does
The GitHub workflow performs the following actions automatically:

Continuous Integration (CI)

- Checks out the GitHub repository.
- Installs the required npm packages used by Azure Data Factory build utilities.
- Validates Data Factory resources.
- Performs ARM template validation.
- Generates deployment-ready ARM templates.
- Publishes ARM templates as build artifacts.

Continuous Deployment (CD)

- Downloads generated deployment artifacts.
- Authenticates with Azure using a Service Principal.
- Deploys ARM templates to the destination resource group.
- Updates the target Azure Data Factory environment.
- Supports environment-specific overrides through deployment parameters and GitHub Secrets.

## Deployment Workflow

### Important Prerequisites:

- Source Azure Data Factory using Git integration
- GitHub repository connected to ADF
- Service Principal with Contributor access
- build folder containing package.json
- ARM template generation enabled

---

#### Step 1 - Create an App Registration on the Azure Portal

> GitHub itself cannot deploy resources to Azure unless it authenticates with Azure. The Service Principal is created in Azure AD (Microsoft Entra ID), and this Service Principal serves as the identity that GitHub uses for authentication.

#### Step 2 - Create a Federated Secret under the App Registration

> Instead of using a Client Secret, this solution uses GitHub OpenID Connect (OIDC) Federation to securely authenticate GitHub Actions with Microsoft Entra ID.>

> Federated Credentials establish a trust relationship between GitHub and Azure, allowing GitHub Actions workflows to obtain short-lived access tokens directly from Microsoft Entra ID without storing credentials in the repository.

> While creating the Federated Credential, select GitHub Actions Deploying Azure Resources as the federated credential scenario. You will then be prompted to provide the following information:

 - Organization → Enter the GitHub organization name
 - Organization ID → Retrieve this from _https://api.github.com/users/<Organization_Name>_ and look for the 'id' field.
 - Repository → Enter the repository name
 - Repository ID → Retrieve this from _https://api.github.com/repos/<Organization_Name>/<Repository_Name>_ and look for the 'id' field.
 - If you receive an error and cannot retrieve the Repository ID → This may occur if you are using a personal GitHub account or if the repository is set to Private. Temporarily change the repository visibility from Private to Public, complete the required configuration, and then change the repository visibility back to Private.
- Entity Type → Select Branch from the dropdown and enter the collaboration branch name in the next field.

##### Benefits of using Federated Credentials:

- Eliminates the need to store Client Secrets in GitHub.
- Reduces the risk of secret leakage or credential exposure.
- Removes secret expiration and rotation management.
- Uses short-lived tokens generated during workflow execution.
- Follows Microsoft's recommended authentication approach for GitHub Actions.

#### Step 3 - Grant Contributor Access

> The Service Principal must have permissions to deploy Azure Data Factory resources.

#### Step 4 - Configure GitHub Repository Secrets

Configure the following GitHub Secrets before running the workflow: 

> Repository → Settings → Secrets and Variables → Actions

| Secret | Description | Location |
|----------|-------------|------------|
| AZURE_CLIENT_ID | Service Principal Client Id | App Registration → Overview → Application (client) ID |
| AZURE_SUBSCRIPTION_ID | Azure Subscription Id | Subscription → Overview |
| AZURE_TENANT_ID | Microsoft Entra Tenant Id | App Registration → Overview → Directory (tenant) ID |
| AZURE_RESOURCEGROUP_NAME | Resource Group Hosting ADF | ADF → Overview |
| SOURCE_DATA_FACTORY | Source ADF Name | ADF → Overview |
| DESTINATION_DATA_FACTORY | Destination ADF Name | ADF → Overview |

#### Step 5 - Build Folder

The build folder contains: _package.json_

> GitHub Actions executes npm commands from this location to validate and export ARM templates.

---

## CICD Deployment Best Practices

#### Development Factory Only for Git Integration
Use Git integration only for the development Data Factory. Test, UAT, and Production environments should receive changes through the deployment pipeline rather than direct source control integration. This ensures consistency and controlled promotion across environments.

#### Automate Trigger Management During Deployments
Triggers may need to be stopped before deployment and restarted after deployment to avoid deployment failures and unintended executions. Microsoft provides deployment scripts that can automate trigger handling and cleanup activities during CI/CD processes.

#### Maintain Consistent Integration Runtime Configuration
When promoting resources across environments, Integration Runtime names, types, and configurations should remain consistent. This is especially important for Self-Hosted Integration Runtime deployments across Development, Test, and Production environments.

#### Parameterize Environment-Specific Settings
Resources such as Data Factory names, Key Vault names, endpoints, and connection configurations should be parameterized to enable seamless deployments across multiple environments.

Store secrets in environment-specific Azure Key Vaults and reference them through deployment parameters. Keeping secret names consistent across environments simplifies deployment and reduces configuration complexity.

#### Include Global Parameters in ARM Templates
Global Parameters should be included in ARM template deployments to ensure configuration consistency across environments and simplify CI/CD implementations. 

#### Follow Consistent Naming Standards
Avoid spaces in ADF resource names. Prefer using underscores (_) or hyphens (-) for improved compatibility and maintainability.

#### Keep the Repository Clean
Only maintain files required for Azure Data Factory source control and deployment. Unnecessary backup or temporary files may lead to repository maintenance issues and deployment complications.

#### Use Controlled Feature Rollout Strategies
When deploying changes that should not immediately execute in higher environments, consider feature toggles or environment-driven logic using Global Parameters and conditional execution patterns. This enables deployment without immediate exposure of new functionality.

#### Support Hotfix Deployments
For urgent production issues, maintain a controlled hotfix deployment process so that critical fixes can be promoted independently without requiring a full release cycle. 


## Known Limitations

#### Selective Deployment Is Not Supported
Azure Data Factory deployments operate on the complete factory metadata. Deployments are intended to promote the entire validated factory state rather than individual resources. Microsoft recommends using a dedicated hotfix process for exceptional production scenarios.

#### Publishing from Non-Collaboration Branches
Publishing and deployment processes are designed around the configured collaboration branch strategy and not private development branches.

#### Resource Dependency Requirements
ADF resources are highly interconnected. Pipelines, datasets, triggers, and linked services have dependency relationships that must remain intact during deployments.
