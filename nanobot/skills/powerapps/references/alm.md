# Solution Management and ALM

## Table of Contents

- [Solutions Overview](#solutions-overview)
- [Solution Components](#solution-components)
- [Environment Strategy](#environment-strategy)
- [Source Control](#source-control)
- [CI/CD Pipelines](#cicd-pipelines)
- [Environment Variables](#environment-variables)
- [Managed vs Unmanaged](#managed-vs-unmanaged)

## Solutions Overview

Solutions are the transport mechanism for Power Platform components. All customizations should be built inside a solution for portability and ALM.

### Creating a Solution

1. Navigate to make.powerapps.com > Solutions > New Solution.
2. Provide display name, publisher prefix, and version.
3. Add or create components inside the solution.

### Publisher and Prefix

Choose a publisher prefix carefully -- it prefixes all schema names and cannot be changed after creation.

```
Publisher: Contoso
Prefix: contoso
Table: contoso_Employee
Column: contoso_HireDate
```

Avoid the default `cr` prefix. Use a short, recognizable prefix (2-5 chars).

## Solution Components

A solution can contain:

| Component | Description |
|-----------|-------------|
| Tables (entities) | Dataverse table definitions, columns, relationships |
| Canvas apps | Power Apps canvas applications |
| Model-driven apps | Power Apps model-driven applications |
| Flows | Power Automate cloud flows |
| Connection references | Abstracted connection configurations |
| Environment variables | Configuration values that vary per environment |
| Security roles | RBAC role definitions |
| Web resources | HTML, JS, CSS, images for model-driven apps |
| Plug-in assemblies | Server-side .NET code |
| Business rules | Declarative logic on Dataverse forms |
| Dashboards and charts | Reporting components |
| Sitemaps | Navigation structure for model-driven apps |

## Environment Strategy

### Recommended Environments

| Environment | Purpose | Solution Type |
|-------------|---------|---------------|
| **Dev** | Active development | Unmanaged |
| **Test / QA** | Testing and validation | Managed |
| **UAT** | User acceptance testing | Managed |
| **Production** | End users | Managed |

### Environment Setup

- Enable Dataverse in each environment.
- Configure security roles per environment.
- Set up connection references and environment variables per environment.
- Use environment groups for consistent governance.

## Source Control

### Power Platform CLI (pac)

The `pac` CLI enables exporting solutions as source files for version control.

```bash
# Authenticate
pac auth create --url https://yourorg.crm.dynamics.com

# Export solution as zip
pac solution export --name YourSolution --path ./exports/YourSolution.zip

# Unpack to source files
pac solution unpack --zipfile ./exports/YourSolution.zip --folder ./src/YourSolution --processCanvasApps

# After making changes, pack back to zip
pac solution pack --folder ./src/YourSolution --zipfile ./build/YourSolution.zip

# Import solution
pac solution import --path ./build/YourSolution.zip
```

### Source File Structure

```
src/YourSolution/
├── solution.xml               # Solution manifest
├── Entities/
│   └── contoso_employee/
│       ├── Entity.xml         # Table definition
│       ├── FormXml/           # Form layouts
│       ├── SavedQueries/      # Views
│       └── Charts/            # Charts
├── CanvasApps/
│   └── contoso_EmployeeApp/   # Unpacked canvas app source
│       ├── Properties.json
│       ├── Screens/
│       └── Components/
├── Workflows/                 # Flows
├── ConnectionReferences/
├── EnvironmentVariableDefinitions/
└── Roles/
```

### Git Branching Strategy

```
main (production-ready)
├── release/v1.2          # release candidate
├── feature/new-form      # feature branch
└── hotfix/fix-delegation # urgent fix
```

- Develop in feature branches.
- Merge to `main` via pull request with review.
- Tag releases.

## CI/CD Pipelines

### Using Power Platform Build Tools (Azure DevOps)

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'windows-latest'

steps:
  - task: PowerPlatformToolInstaller@2
    displayName: 'Install Power Platform Build Tools'

  - task: PowerPlatformPackSolution@2
    displayName: 'Pack Solution'
    inputs:
      SolutionSourceFolder: 'src/YourSolution'
      SolutionOutputFile: '$(Build.ArtifactStagingDirectory)/YourSolution.zip'
      SolutionType: 'Managed'

  - task: PowerPlatformImportSolution@2
    displayName: 'Import to Target'
    inputs:
      authenticationType: 'PowerPlatformSPN'
      PowerPlatformSPN: 'YourServiceConnection'
      SolutionInputFile: '$(Build.ArtifactStagingDirectory)/YourSolution.zip'
      AsyncOperation: true
      MaxAsyncWaitTime: 60
```

### Using GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy Solution
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install PAC CLI
        uses: microsoft/powerplatform-actions/install-pac@v1

      - name: Authenticate
        uses: microsoft/powerplatform-actions/who-am-i@v1
        with:
          environment-url: ${{ secrets.POWER_PLATFORM_URL }}
          app-id: ${{ secrets.CLIENT_ID }}
          client-secret: ${{ secrets.CLIENT_SECRET }}
          tenant-id: ${{ secrets.TENANT_ID }}

      - name: Pack Solution
        uses: microsoft/powerplatform-actions/pack-solution@v1
        with:
          solution-folder: 'src/YourSolution'
          solution-file: 'build/YourSolution_managed.zip'
          solution-type: 'Managed'

      - name: Import Solution
        uses: microsoft/powerplatform-actions/import-solution@v1
        with:
          environment-url: ${{ secrets.POWER_PLATFORM_URL }}
          app-id: ${{ secrets.CLIENT_ID }}
          client-secret: ${{ secrets.CLIENT_SECRET }}
          tenant-id: ${{ secrets.TENANT_ID }}
          solution-file: 'build/YourSolution_managed.zip'
          force-overwrite: true
          run-asynchronously: true
```

### Service Principal Setup

For automated deployments, register an app in Azure AD:

1. Register application in Azure AD.
2. Create client secret.
3. In Power Platform Admin Center, add the app as an application user.
4. Assign security role (System Administrator or custom).
5. Store credentials as pipeline secrets.

## Environment Variables

Environment variables store configuration that varies per environment (URLs, feature flags, IDs).

### Types

| Type | Use Case |
|------|----------|
| Text | API URLs, config strings |
| Number | Thresholds, limits |
| Yes/No | Feature flags |
| JSON | Complex configuration objects |
| Data Source | Connection reference to a data source |
| Secret | Reference to Azure Key Vault secret |

### Defining in Solution

```
// In solution: New > Environment Variable
Name: contoso_APIEndpoint
Display Name: API Endpoint
Type: Text
Default Value: https://api-dev.example.com
Current Value: (set per environment)
```

### Using in Power Fx

```
// Canvas app: look up environment variable value
LookUp(
    'Environment Variable Values',
    'Environment Variable Definition'.'Schema Name' = "contoso_APIEndpoint"
).'Value'

// Or use in a flow and pass to app as parameter
```

### Using in Flows

```
// In Power Automate:
// Add action: "List rows" from Environment Variable Values table
// Or reference directly in expressions:
// @{outputs('Get_Env_Var')?['body/value']}
```

## Managed vs Unmanaged

### Unmanaged Solutions

- Used in **development** environments.
- Components can be edited directly.
- Can be layered -- last-edited wins.
- No deletion tracking.
- Export as unmanaged for further development.

### Managed Solutions

- Used in **test, UAT, and production** environments.
- Components are locked -- cannot be edited in target environment.
- Clean uninstall -- removing solution removes all components.
- Supports solution layering and segmentation.
- Recommended for all non-development deployments.

### Solution Layering

```
Layer 3: Your Managed Solution (top -- wins conflicts)
Layer 2: Another ISV Solution
Layer 1: System Solution (base Dataverse)
```

Higher layers override lower layers. Active customizations (unmanaged) always sit on top.

### Versioning

Use semantic versioning: `MAJOR.MINOR.BUILD.REVISION`

```
1.0.0.0  -- initial release
1.1.0.0  -- new features (minor)
1.1.1.0  -- bug fixes (build)
2.0.0.0  -- breaking changes (major)
```

Increment version before each export. The `pac` CLI can automate this:

```bash
pac solution online-version --solution-name YourSolution --solution-version 1.2.0.0
```

### Solution Segmentation

For large solutions, consider splitting into:

- **Core**: shared tables, security roles, base configuration.
- **App**: individual apps and their specific flows.
- **Integration**: connectors, flows for external systems.

Each segment is a separate solution with dependencies declared.
