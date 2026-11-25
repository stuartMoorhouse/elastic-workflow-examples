# Elastic Workflow Examples

A collection of sample [Elastic Workflows](https://www.elastic.co/docs/solutions/security/get-started/workflows)
for Elastic Security use cases: alert triage, case management, cross-SIEM
sync, SOC reporting, and a Red Team exercise pipeline.

## What's here

See [`workflows/`](workflows/) for the full list, or jump straight to a folder:

| Workflow | Team | What it does |
|---|---|---|
| [`siem-alert-triage`](workflows/siem-alert-triage/) | Blue | SIEM alert → AI vulnerability analysis → fleet-wide osquery scan → AI incident report → Security Case |
| [`okta-credential-stuffing`](workflows/okta-credential-stuffing/) | Blue | Okta account-takeover alert → AI-written case → remediation script on the compromised endpoint → case closed |
| [`password-spray-response`](workflows/password-spray-response/) | Blue | Password-spray alert → Security Case → AI analysis comment (analysis only, no remediation) |
| [`sentinel-alert-status-sync`](workflows/sentinel-alert-status-sync/) | Blue | Keeps alert status in sync between Elastic Security and Microsoft Sentinel every minute |
| [`situation-report`](workflows/situation-report/) | Blue | Weekly TLP-marked SOC situation report generated from Security Timeline notes |
| [`reaper-red-team-pipeline`](workflows/reaper-red-team-pipeline/) | Red | Full automated kill chain: recon → AI attack planning → exploitation → privesc → AI assessment report |

## Using a workflow

Each folder has:
- `workflow.yaml` the workflow definition, with header comments on what it
  does and what it depends on (indices, agents, connectors, Fleet, external
  services).
- `SETUP.md` step-by-step instructions for creating the test data,
  connectors, Agent Builder agents, and (where relevant) detection rules
  needed to run it against a real Elastic Security cluster and confirm it
  works.


## Scope and caveats

These are for demonstration purposes only, not intended for Production use, and not official Elastic software. 
