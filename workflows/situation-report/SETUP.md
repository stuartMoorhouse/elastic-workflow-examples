# Setup: Weekly Situation Report

Follow this to stand up test data in an Elastic Security cluster and run
`workflow.yaml` end to end.

## 1. Create the Kibana connector

Stack Management > Connectors > create a Generative AI connector (Elastic
Managed LLM on Elastic Cloud, or your own Anthropic/OpenAI/Bedrock
connector). Note its connector id.

In `workflow.yaml`, replace:
- `__CONNECTOR_ID__` with that connector id
- `__MODEL__` with a label for whatever model it's using, e.g. `"claude-sonnet-4-6"`
- `__PERIOD_DAYS__` with a look-back window, e.g. `7`

## 2. Create test Timelines with notes

The workflow reads **Security Timeline saved objects** and their **notes**,
not a plain Elasticsearch index. Create these through Kibana's UI
(Security > Investigations > Timelines) or the Saved Objects API.

Timeline titles must follow this convention (the workflow parses it):

```
YYYY-MM-DD | <client or asset> | <technique or rule> | <status> | TLP:<colour>
```

### Sample Timeline 1 — qualifies for the report

**Title:**
```
2026-11-18 | CLIENT-B | T1547.001 Registry Run Key | true-positive | TLP:AMBER
```

**Add one note to this timeline** (Timeline > Notes, or a global note) with
this exact body — the workflow reads IOCs, ATT&CK ids and the trailing
`Keywords:` line out of this text, so the format matters:

```
Investigated a suspicious Run key added under
HKCU\Software\Microsoft\Windows\CurrentVersion\Run on host WIN-FIN-02.
Payload dropped at C:\Users\jsmith\AppData\Roaming\update.exe,
SHA256 3f786850e387550fdab836ed7e6dc881de23001b.
Beaconed to 203.0.113.44 over HTTPS every 60s. Account jsmith@clientb.com
was used for initial access via a phished credential. Technique:
T1547.001 (Boot or Logon Autostart Execution: Registry Run Keys).
Status: true positive, contained — host isolated, key removed, account
password reset. Not reportable under NIS2 (contained within 2 hours,
no data exfiltration confirmed).

Keywords: true-positive registry-persistence t1547.001
```

### Sample Timeline 2 — out of period / wrong type, should be excluded

**Title:**
```
2026-01-05 | CLIENT-A | Generic alert | false-positive | TLP:CLEAR
```
No note needed — its date is outside the default 7-day window, so it's a
good check that the workflow actually filters by period.

### Sample Timeline 3 — a template, must always be excluded

Create one Timeline and save it **as a template** (Kibana's "Save as
template" option sets `timelineType: template`). Give it any title — the
workflow explicitly skips `timelineType: template` regardless of title or
date, so this checks that filter works.

## 3. Create the output index (optional but recommended)

```
PUT situation-reports
{
  "mappings": {
    "properties": {
      "title":          { "type": "text" },
      "tlp":            { "type": "keyword" },
      "client":         { "type": "keyword" },
      "period_start":   { "type": "date", "format": "yyyy-MM-dd" },
      "period_end":     { "type": "date", "format": "yyyy-MM-dd" },
      "body":           { "type": "text" },
      "sources":        { "type": "keyword" },
      "keywords":       { "type": "keyword" },
      "filter_client":  { "type": "keyword" },
      "filter_keywords":{ "type": "keyword" },
      "no_data":        { "type": "boolean" },
      "generated_at":   { "type": "date" },
      "model":          { "type": "keyword" },
      "workflow_execution_id": { "type": "keyword" }
    }
  }
}
```
If you skip this, the first run dynamically maps the index, which is fine
for testing but may map `client`/`tlp` as `text` instead of `keyword`.

## 4. Enable Cases

Confirm Security > Cases is enabled in the space you're testing in (it is by
default on a standard Security deployment).

## 5. Run it

Upload `workflow.yaml` to Kibana (Workflows), then trigger it manually with
empty `client`/`keywords` inputs first.

**Expected result**, with the sample data above:
- `get_timelines` returns Timeline 1 (and Timeline 2, since the API call
  itself isn't period-filtered — filtering by date happens in the prompt).
- The generated report's TLP is `TLP:AMBER`, client is `CLIENT-B`, and the
  Indicators of compromise table includes the IP, hash, and file path from
  the note.
- A document lands in `situation-reports`.
- A new Security Case is created, owned by `securitySolution`, tagged
  `situation-report`, `TLP:AMBER`, and `scope:2026-11-24:all:any` (today's
  date as period_end, assuming the 7-day window run today).

Then re-run with `inputs.client = "CLIENT-B"` to confirm the `search=`
filter narrows to just that timeline, and run once with no qualifying
Timelines at all (e.g. after deleting them) to confirm `no_data: true` is
handled — no case should be created or updated in that case.
