# Elastic Workflow Examples

A collection of sample [Elastic Workflows](https://www.elastic.co/docs/solutions/security/get-started/workflows)
for Elastic Security use cases — alert triage, case management, cross-SIEM
sync, SOC reporting, and a red team exercise pipeline.

Each workflow was extracted from a working demo repo, then documented so it
can be understood and tested on its own, without the rest of that repo.

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
- `workflow.yaml` — the workflow definition, with header comments on what it
  does and what it depends on (indices, agents, connectors, Fleet, external
  services).
- `SETUP.md` — step-by-step instructions for creating the test data,
  connectors, Agent Builder agents, and (where relevant) detection rules
  needed to run it against a real Elastic Security cluster and confirm it
  works.

Start with a folder's `SETUP.md`, not just the YAML — several of these
workflows depend on an Agent Builder agent or a detection rule that has to
be created by hand first; the YAML alone won't run.

## Scope and caveats

These are sample/demo workflows, not production-hardened ones:

- Agent system prompts in each `SETUP.md` are drafted starting points, not
  verified configurations — test and tune them before relying on the output.
- `reaper-red-team-pipeline` actively attacks its target (port scanning,
  exploitation, credential brute-forcing, privilege escalation). Only ever
  point it at a host you have explicit written authorization to test.
- No secrets live in this repo. Every connector id, API key, client secret,
  and tenant/subscription id is a placeholder (`REPLACE_WITH_...`,
  `__CONNECTOR_ID__`, `<tenant-id>`) to be filled in locally, not committed.
