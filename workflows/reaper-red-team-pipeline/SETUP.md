# Setup: Reaper Red Team Pipeline

**Authorization required.** This pipeline actively port-scans, exploits, and
brute-forces a target, then escalates privileges on it. Only ever point it
at a host you have explicit written authorization to test — your own lab
VM or a CTF/range box you control. Do not point `target`/`consts.target_ip`
at anything else.

This is the hardest of the six to make self-contained: the 4 HTTP services
it calls aren't Elastic products, and this repo doesn't include their
implementation — only the contract this workflow expects from them. You
need to stand those up yourself (or stub them) before this is testable.

## 1. Stand up a target

A deliberately-vulnerable VM is the standard choice for this kind of test,
e.g. Metasploitable2, or any CTF box you're authorized to attack. Note its
IP for `consts.target_ip` / `inputs.target`.

## 2. Stand up (or stub) the four HTTP services

Each needs to accept `X-Webhook-Secret: <consts.webhook_secret>` and return
JSON under a `data` key. For an initial smoke test of the **workflow
wiring** (not the actual attack tooling), stub each as a minimal HTTP
service returning canned data matching the shape `workflow.yaml`'s
templates expect:

```
POST <nmap_url>/scan        { target, options }
  -> { "data": { "hosts": [{"ip": "...", "ports": [{"port": 22, "service": "ssh", "version": "OpenSSH 7.2p2"}]}] } }

POST <nuclei_url>/scan      { target }
  -> { "data": { "findings": [{"cve": "CVE-2023-38408", "severity": "critical"}] } }

POST <hydra_url>/attack     { target, nmap_hosts }
  -> { "data": { "cracked": [{"service": "ssh", "user": "admin", "password": "..."}] } }

POST <metasploit_url>/exploit  { target, nmap_hosts, nuclei_findings }
  -> { "data": { "session_id": "...", "is_root": false, "uid_output": "uid=1000(user)", "platform": "linux", "session_type": "shell" } }
POST <metasploit_url>/linpeas   { session_id }
  -> { "data": { "findings": ["world-writable /etc/cron.d/backup.sh owned by root"] } }
POST <metasploit_url>/privesc   { session_id, linpeas_findings }
  -> { "data": { "session_id": "...", "is_root": true } }
```

For a *real* run against your authorized target, these need to be thin
wrapper APIs in front of actual nmap/nuclei/hydra/metasploit-framework
installs — not provided here.

## 3. Set the webhook secret

Pick a shared secret, put it in `consts.webhook_secret`, and configure each
of the 4 services to require/check the same value on the
`X-Webhook-Secret` header.

## 4. Create the three Agent Builder agents

Each needs an LLM connector. Starter instructions:

### `reaper-attack-planner`
```
You are an offensive security planner. You will be given NMAP, Nuclei and
Hydra results as JSON.

Propose the single best exploitation approach: which finding to exploit,
which tool/module, and why. Prefer an approach that lands as a NON-ROOT
user, to demonstrate the full kill chain through a later privilege
escalation step. State your plan in plain text, referencing the specific
CVE/service/credential you're using.
```

### `reaper-privesc-planner`
```
You are an offensive security planner focused on privilege escalation. You
will be given the current session's user/platform/session type and LinPEAS
enumeration findings as JSON.

Recommend ONE concrete privilege-escalation path to root, naming the
specific misconfiguration/vulnerability from the LinPEAS findings and the
technique (e.g. SUID binary, writable cron job, kernel exploit) to use.
```

### `reaper-report-writer`
```
You are a red team report writer. You will be given the full output of a
kill-chain engagement: recon (NMAP/Nuclei/Hydra), the attack plan, initial
access result, LinPEAS enumeration, the privesc plan, and the final
privilege-escalation result.

Write a red team assessment report using this template:

# Red Team Assessment: <target>

## Executive Summary
What was achieved (initial access, privilege level reached) and the
business risk this demonstrates.

## Reconnaissance
Open ports/services (NMAP) and vulnerabilities found (Nuclei).

## Initial Access
How access was obtained, as which user.

## Privilege Escalation
The path used to reach root, and the misconfiguration behind it.

## Recommendations
2-4 concrete remediation steps, most urgent first.

Only use information you were given — never invent findings.
```

## 5. Run it

Trigger manually with `inputs.target` set to your authorized test VM's IP
(or rely on `consts.target_ip` if you set it there instead).

**Expected result (with the stubbed services from step 2):** each phase
step populates in order; `linpeas_enum`/`privesc_planner`/
`metasploit_privesc` only run because the stub's `is_root` was `false`;
`generate_report` produces a Markdown report referencing the CVE and
privesc technique from your stub data. Swap in real tooling once the
wiring is confirmed.
