# Setup: Elastic to Sentinel Alert Status Sync

This one has a real external dependency (a Microsoft Sentinel workspace) —
there's no way to fully fake that locally, but here's the minimum to test
the sync logic itself.

## 1. Azure side

1. Create (or use an existing) Log Analytics workspace with Microsoft
   Sentinel enabled.
2. Register a Microsoft Entra ID app:
   ```
   az ad app create --display-name elastic-workflow-sentinel-sync
   az ad sp create --id <app-id>
   az ad app credential reset --id <app-id>   # note the client secret
   ```
3. Assign it the **Microsoft Sentinel Responder** role, scoped to the
   workspace:
   ```
   az role assignment create \
     --assignee <app-id> \
     --role "Microsoft Sentinel Responder" \
     --scope /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.OperationalInsights/workspaces/<workspace>
   ```
4. Note the ARM incidents URL for that workspace:
   ```
   https://management.azure.com/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.OperationalInsights/workspaces/<workspace>/providers/Microsoft.SecurityInsights/incidents
   ```

## 2. Create a test Sentinel incident

Easiest path: in the Sentinel portal, create an analytics rule that always
fires on a dummy log entry, or just create an incident manually via the
Sentinel UI/API. Note its **incident name** (the GUID in its resource path,
not its title) — this is what `azure.eventhub.IncidentName` must equal.

```
az rest --method put \
  --url "https://management.azure.com/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.OperationalInsights/workspaces/<workspace>/providers/Microsoft.SecurityInsights/incidents/<new-guid>?api-version=2023-02-01" \
  --body '{"properties": {"title": "Test incident for sync", "severity": "Medium", "status": "New"}}'
```

## 3. Create the Kibana HTTP connector

Stack Management > Connectors > HTTP connector, OAuth2 client credentials:
```
URL:              https://management.azure.com
Auth type:        OAuth 2.0 client credentials
Access token URL: https://login.microsoftonline.com/<tenant-id>/oauth2/v2.0/token
Client ID:        <the Entra app's client id>
Scope:            https://management.azure.com/.default
Client secret:    <the Entra app's client secret>
```
Note the connector id.

## 4. Fill in the workflow placeholders

In `workflow.yaml`, replace:
- `consts.incidents_url` — the ARM incidents URL from step 1.4
- `kibana.alert.rule.name` filter value — the name of the detection rule
  below
- both `REPLACE_WITH_AZURE_ARM_CONNECTOR_ID` — the connector id from step 3

## 5. Create the detection rule + sample alert

Index a synthetic alert carrying the Sentinel incident's GUID, and a rule
that just matches it (a real deployment would ingest actual Sentinel
incidents via Event Hub and build the rule over that — faking the end
document is enough to test the sync workflow in isolation):

```json
PUT .alerts-security.alerts-default/_doc/test-alert-1
{
  "kibana.alert.rule.name": "Microsoft Sentinel Incident Raised",
  "kibana.alert.workflow_status": "closed",
  "kibana.alert.workflow_status_updated_at": "2026-11-25T14:02:00.000Z",
  "kibana.alert.workflow_reason": "true_positive",
  "azure.eventhub.IncidentName": "<the incident GUID from step 2>"
}
```
(Direct writes to the hidden `.alerts-*` index aren't normally supported —
in practice, trigger this by actually acknowledging/closing a real alert
from a rule whose data includes `azure.eventhub.IncidentName`, e.g. a rule
over data ingested through the Event Hub pipeline in the
`elastic-sentinel-integration` repo. Use the direct-write approach only
against a disposable test cluster.)

## 6. Run it

Either wait for the `1m` schedule or trigger the workflow manually.

**Expected result:** the Sentinel incident from step 2 flips to `status:
Closed` with `classification: TruePositive`, `classificationReason:
SuspiciousActivity`. Confirm with:
```
az rest --method get --url ".../incidents/<guid>?api-version=2023-02-01"
```
Re-run with `kibana.alert.workflow_status: "acknowledged"` and no
`workflow_reason` to confirm the incident instead goes to `status: Active`
with no classification set.
