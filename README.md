# azure-functions-action

Composite GitHub Actions for Azure Function App deployment lifecycle: build and deploy a .NET function app, check health via Application Insights, and roll back automatically if needed.

> **Prerequisite:** Basic auth must be disabled on the Function App — these actions authenticate via Azure AD bearer token. The `save`, `check-health`, and `rollback` actions require a prior `azure/login@v2` step; `deploy-code` handles login internally.

---

## Actions

### `StottHoare/azure-functions-action/save@main`

Downloads the current deployed `wwwroot` as `previous.zip` for potential rollback, and records the time of the save as `deploy-time`.

If the download fails or returns an empty file (e.g. on a first deploy), a warning is logged and the action continues — rollback simply won't be available.

**Inputs**

| Name | Required | Description |
|------|----------|-------------|
| `app-name` | yes | Full Azure Function App name, e.g. `shd-DataGateway-func` |
| `resource-group` | yes | Azure resource group containing the Function App |

**Outputs**

| Name | Description |
|------|-------------|
| `deploy-time` | UTC timestamp recorded before the download, e.g. `2024-01-15T10:30:00Z`. Pass this as `start-time` to `check-health` to exclude pre-deployment metrics. |

---

### `StottHoare/azure-functions-action/check-health@main`

Queries the `azure.functions.health_check.reports` custom metric in Application Insights. Waits 10 seconds for the function host to restart, then retries up to 6 times (30 seconds apart) before reporting a result.

A `valueMax >= 1` indicates a healthy probe. If no data is returned after all attempts, `healthy` is set to `false`.

**Inputs**

| Name | Required | Description |
|------|----------|-------------|
| `app-name` | yes | Full Azure Function App name |
| `resource-group` | yes | Azure resource group. The App Insights resource name is derived by replacing `-rg` with `-appi` (e.g. `shd-prod-rg` → `shd-prod-appi`) |
| `start-time` | yes | ISO8601 UTC timestamp. Only metric data after this time is considered, preventing pre-deployment readings from causing a false pass. Use the `deploy-time` output from `save`. |

**Outputs**

| Name | Description |
|------|-------------|
| `healthy` | `true` or `false` |

---

### `StottHoare/azure-functions-action/rollback@main`

Re-deploys `previous.zip` to the Function App via Kudu zip deploy. If `previous.zip` does not exist or is empty, the action logs a message and exits successfully without doing anything.

Fails if the Kudu API returns a status other than `200` or `202`.

**Outputs**

| Name | Description |
|------|-------------|
| `rollback-time` | UTC timestamp recorded before the rollback deploy. Pass this as `start-time` to a subsequent `check-health` call. |

**Inputs**

| Name | Required | Description |
|------|----------|-------------|
| `app-name` | yes | Full Azure Function App name |

### `StottHoare/azure-functions-action/test-code@main`

Builds a .NET project and optionally runs tests, publishing results via `dorny/test-reporter`.

**Caller responsibilities:**
- Run `actions/checkout@v4` (with submodules if needed) before calling this action
- Set `permissions: id-token: write, packages: read, contents: read, checks: write` on the job
- Set `environment:` on the job if required

**Inputs**

| Name | Required | Description |
|------|----------|-------------|
| `dotnet-version` | yes | .NET version e.g. `8.0`, `9.0`, `10.0` |
| `run-tests` | yes | Whether to run tests (`'true'` or `'false'`) |
| `csproj-path` | yes | Relative path to the `.csproj` to build |
| `environment` | yes | Environment name e.g. `Development`, `Production`. Passed to tests via `DOTNET_ENVIRONMENT` and `AZURE_FUNCTIONS_ENVIRONMENT`. |
| `github-action-client-id` | yes | Client ID of the GitHub Actions service principal |
| `subscription-id` | yes | Azure Subscription ID |
| `tenant-id` | yes | Azure Tenant ID |
| `packages-token` | yes | Token for authenticating to the GitHub Packages NuGet feed |
| `app-configuration-connection-string` | yes | App Configuration connection string passed to tests |
| `cache` | no | Whether to cache NuGet packages in `setup-dotnet`. Defaults to `true`. |

**Example**

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    environment: Development
    permissions:
      id-token: write
      packages: read
      contents: read
      checks: write
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true
          token: ${{ secrets.SUBMODULES_TOKEN }}

      - uses: StottHoare/azure-functions-action/test-code@main
        with:
          dotnet-version: '8.0'
          run-tests: 'true'
          csproj-path: src/DataGateway/DataGateway.Tests.csproj
          environment: Development
          github-action-client-id: ${{ vars.AZURE_APPREG_GITHUBACTIONS_CLIENT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          packages-token: ${{ secrets.PACKAGES_TOKEN }}
          app-configuration-connection-string: ${{ secrets.APP_CONFIGURATION_CONNECTION_STRING }}
```

---

### `StottHoare/azure-functions-action/deploy-code@main`

The all-in-one action. Builds a .NET project, deploys it to a Function App, checks health, and rolls back automatically if the health check fails. Uses the `save`, `check-health`, and `rollback` actions internally.

**Caller responsibilities:**
- Run `actions/checkout@v4` (with submodules if needed) before calling this action
- Set `permissions: id-token: write, packages: read, contents: read` on the job
- Set `environment:` on the job if required

**Inputs**

| Name | Required | Description |
|------|----------|-------------|
| `function-name` | yes | Function resource name excluding environment prefix, e.g. `DataGateway-func` |
| `dotnet-version` | yes | .NET version e.g. `8.0`, `9.0`, `10.0` |
| `csproj-path` | yes | Relative path to the `.csproj` to build |
| `packages-token` | yes | Token for authenticating to the GitHub Packages NuGet feed |
| `github-action-client-id` | yes | Client ID of the GitHub Actions service principal |
| `subscription-id` | yes | Azure Subscription ID |
| `tenant-id` | yes | Azure Tenant ID |
| `resource-prefix` | no | Resource prefix e.g. `shd` or `shp`. Derived from branch name if omitted (`main` → `shp`, else `shd`) |
| `cache` | no | Whether to cache NuGet packages in `setup-dotnet`. Defaults to `true`. |

**Example**

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: Production
    permissions:
      id-token: write
      packages: read
      contents: read
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true
          token: ${{ secrets.SUBMODULES_TOKEN }}

      - uses: StottHoare/azure-functions-action/deploy-code@main
        with:
          function-name: DataGateway-func
          dotnet-version: '8.0'
          csproj-path: src/DataGateway/DataGateway.csproj
          packages-token: ${{ secrets.PACKAGES_TOKEN }}
          github-action-client-id: ${{ vars.AZURE_APPREG_GITHUBACTIONS_CLIENT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
```

---

## Usage (individual actions)

The typical pattern is: save → deploy → check health → rollback if unhealthy.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Azure login
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Save current deployment
        id: save
        uses: StottHoare/azure-functions-action/save@main
        with:
          app-name: shd-DataGateway-func
          resource-group: shd-prod-rg

      - name: Deploy
        # your existing deployment step, e.g. azure/functions-action@v2
        uses: azure/functions-action@v2
        with:
          app-name: shd-DataGateway-func
          package: ./publish

      - name: Check health
        id: health
        uses: StottHoare/azure-functions-action/check-health@main
        with:
          app-name: shd-DataGateway-func
          resource-group: shd-prod-rg
          start-time: ${{ steps.save.outputs.deploy-time }}

      - name: Rollback if unhealthy
        id: rollback
        if: steps.health.outputs.healthy == 'false'
        uses: StottHoare/azure-functions-action/rollback@main
        with:
          app-name: shd-DataGateway-func

      - name: Check health after rollback
        if: steps.health.outputs.healthy == 'false'
        id: health-after-rollback
        uses: StottHoare/azure-functions-action/check-health@main
        with:
          app-name: shd-DataGateway-func
          resource-group: shd-prod-rg
          start-time: ${{ steps.rollback.outputs.rollback-time }}

      - name: Fail if rollback also unhealthy
        if: steps.health.outputs.healthy == 'false' && steps.health-after-rollback.outputs.healthy == 'false'
        run: |
          echo "::error::Deployment and rollback both failed health checks"
          exit 1
```

---

## App Insights naming convention

`check-health` derives the Application Insights resource name from the resource group by replacing the `-rg` suffix with `-appi`:

| Resource group | App Insights |
|----------------|-------------|
| `shd-prod-rg` | `shd-prod-appi` |
| `shd-DataGateway-rg` | `shd-DataGateway-appi` |

If your naming convention differs, raise an issue or open a PR to add an explicit `app-insights-name` input.
