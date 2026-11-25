# Elastic Workflow Examples — Red/Blue Team

Elastic Workflow definitions extracted from Stuart Moorhouse's own repos
(not the `elastic-demo-starter` template), with comments added so each one
can be understood and tested without the surrounding repo.

Each workflow is in its own folder:

```
<workflow-name>/
  workflow.yaml   the workflow definition, with header comments explaining
                  what it does and what it depends on
  SETUP.md        step-by-step: how to create the test data, connectors,
                  agents and (where relevant) detection rules needed to
                  run it in a real Elastic Security cluster and confirm
                  it works
```

| Folder | Team | What it does |
|---|---|---|
| [`siem-alert-triage`](siem-alert-triage/) | Blue | SIEM alert → AI vuln analysis → fleet-wide osquery scan → AI incident report → Security Case |
| [`okta-credential-stuffing`](okta-credential-stuffing/) | Blue | Okta account-takeover alert → AI-written case → remediation script on the compromised endpoint → case closed |
| [`password-spray-response`](password-spray-response/) | Blue | Password-spray alert → Security Case → AI analysis comment (analysis only, no remediation) |
| [`sentinel-alert-status-sync`](sentinel-alert-status-sync/) | Blue | Keeps alert status in sync between Elastic Security and Microsoft Sentinel every minute |
| [`situation-report`](situation-report/) | Blue | Weekly TLP-marked SOC situation report generated from Security Timeline notes |
| [`reaper-red-team-pipeline`](reaper-red-team-pipeline/) | Red | Full automated kill chain: recon → AI attack planning → exploitation → privesc → AI assessment report |

Every workflow that calls an AI agent (`siem-alert-triage`,
`okta-credential-stuffing`, `reaper-red-team-pipeline`) needed system
prompts drafted for its Agent Builder agents — none existed in the source
repos. Those are in each `SETUP.md`; treat them as untested starting
points, not verified prompts.
