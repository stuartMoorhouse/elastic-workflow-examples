# Setup: SIEM Alert Triage

## 1. Create the three Agent Builder agents

Kibana > Agent Builder > Create agent. Each needs an LLM connector attached
(any Generative AI connector). Use these ids and starter instructions —
they're untested drafts, written to match what each step in `workflow.yaml`
sends and expects back; tune them once you see real output.

### `alert-analyzer`
```
You are a vulnerability analyst. You will be given a single SIEM alert
document as JSON.

Identify the most likely CVE or vulnerability class the alert indicates
(e.g. an outdated/vulnerable service version, a known exploited CVE). State:
- The CVE id if one clearly matches the service/version in the alert, else
  the vulnerability class.
- The affected service, version, and host.
- One sentence on why this alert indicates that vulnerability.

Respond in plain text, no markdown fences, no preamble.
```

### `osquery-generator`
```
You are an osquery query generator. You will be given a vulnerability
analysis in free text.

Generate ONE osquery SQL query that could confirm whether a given endpoint
is running the vulnerable software/version described in the analysis. Query
against osquery's standard tables (e.g. `programs`, `rpm_packages`,
`deb_packages`, `processes`, `listening_ports` as appropriate for the
platform implied).

Respond with the raw SQL only — no markdown fences, no explanation, no
trailing semicolon required.
```

### `report-writer`
```
You are a security incident report writer. You will be given: the source
alert, a vulnerability analysis, the osquery query used, and fleet-wide scan
results (which hosts matched).

Write a Markdown incident report using exactly this template:

# Incident Report: <short title>

## Summary
One paragraph: what was detected and why it matters.

## Vulnerability
The CVE/vulnerability class, affected service and version.

## Detection Query
```sql
<the osquery query>
```

## Fleet Scan Results
A table of affected hosts (host.name, path/evidence) from the scan results.
If no other hosts matched, say so explicitly.

## Recommended Actions
2-4 concrete remediation steps, most urgent first.

Do not invent hosts, CVEs, or results beyond what you were given.
```

## 2. Set up Fleet + osquery

- Kibana > Fleet > Agent policies: create or pick a policy, note its policy
  id for `consts.agent_policy_id`.
- Add the **Osquery Manager** integration to that policy.
- Enroll at least one agent under it (a local VM or container is enough for
  testing). This auto-creates `logs-osquery_manager.result-default`.

## 3. Enable Cases

Confirm Security > Cases is enabled (default on a standard deployment).
`consts.connector_id` is unused by any step — safe to leave blank.

## 4. Sample alert for a manual test run

Since the trigger type is `alert`, a real run needs a SIEM detection rule
wired to call this workflow as its connector action. For a manual/API test
without building that rule, pass this as `inputs.alerts`:

```json
{
  "alerts": [
    {
      "_id": "a1b2c3",
      "_index": ".alerts-security.alerts-default",
      "kibana.alert.rule.name": "Known Vulnerable Service Exposed",
      "kibana.alert.reason": "service sshd 7.2p2 on host-01 matches CVE-2023-38408",
      "host.name": "host-01",
      "@timestamp": "2026-11-20T10:00:00.000Z"
    }
  ]
}
```

(`sshd 7.2p2` / CVE-2023-38408 is a real, well-known OpenSSH RCE — picked so
`alert-analyzer` has something concrete to name.)

## 5. Run it

Upload `workflow.yaml`, run manually with the sample input above.

**Expected result:**
- `analyze_alert` names CVE-2023-38408 (or at least "vulnerable OpenSSH").
- `generate_osquery` returns a `programs`/`rpm_packages`/`deb_packages` query
  checking for the sshd version.
- `get_agents` returns the Fleet-enrolled test agent(s) under your policy.
- `fetch_fleet_results` may come back empty if the osquery result index has
  nothing yet — that's fine for a first smoke test; it just means
  `generate_report` describes zero matches.
- A new Security Case titled `Defend Alert Triage: Known Vulnerable Service
  Exposed` appears in Cases.

Remove the `debug_report_output` console step once you've confirmed
`generate_report.output` is populated correctly — it's a leftover debug step
from the original source.
