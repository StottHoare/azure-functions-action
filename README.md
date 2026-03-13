# azure-functions-action

Composite GitHub Actions for Azure Function App deployment lifecycle: save the current deployment, check health via Application Insights, and roll back if needed.

> **Prerequisite:** All actions require a prior `azure/login@v2` step in your workflow. Basic auth must be disabled on the Function App — these actions authenticate via Azure AD bearer token.

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

---

## Usage

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
