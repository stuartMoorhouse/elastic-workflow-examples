# Setup: Okta Credential Stuffing Response

## 1. Install the Okta integration

Kibana > Fleet > Integrations > add the Okta integration (or any ingestion
path that writes to `logs-okta.system-default`). For testing, you can skip
real Okta connectivity and bulk-index synthetic documents directly instead
(see step 2).

## 2. Sample data — the four-stage attack chain

This workflow is triggered by a **detection rule you need to build**: it
must correlate, for the same `okta.actor.alternate_id` + `okta.client.ip`,
a sequence of: repeated failed logins → MFA failures → a successful login →
a post-compromise action. Index this sequence of documents into
`logs-okta.system-default` to give such a rule (and this workflow) something
to fire on:

```json
// Stage 1 — failed logins (repeat ~6x with increasing @timestamp)
{
  "@timestamp": "2026-11-20T09:55:00.000Z",
  "event.action": "user.session.start",
  "okta.event_type": "user.session.start",
  "okta.outcome.result": "FAILURE",
  "okta.outcome.reason": "INVALID_CREDENTIALS",
  "okta.actor.alternate_id": "j.doe@example.com",
  "okta.client.ip": "203.0.113.7"
}

// Stage 2 — MFA failures (repeat ~3x)
{
  "@timestamp": "2026-11-20T09:57:00.000Z",
  "event.action": "user.mfa.factor.deny_push",
  "okta.event_type": "user.mfa.factor.deny_push",
  "okta.outcome.result": "FAILURE",
  "okta.actor.alternate_id": "j.doe@example.com",
  "okta.client.ip": "203.0.113.7"
}

// Stage 3 — successful login
{
  "@timestamp": "2026-11-20T09:58:00.000Z",
  "event.action": "user.session.start",
  "okta.event_type": "user.session.start",
  "okta.outcome.result": "SUCCESS",
  "okta.actor.alternate_id": "j.doe@example.com",
  "okta.client.ip": "203.0.113.7"
}

// Stage 4 — post-compromise action
{
  "@timestamp": "2026-11-20T09:59:30.000Z",
  "event.action": "user.account.privilege.grant",
  "okta.event_type": "user.account.privilege.grant",
  "okta.outcome.result": "SUCCESS",
  "okta.actor.alternate_id": "j.doe@example.com",
  "okta.client.ip": "203.0.113.7"
}
```

### Minimal detection rule to produce a matching alert

A custom query rule (KQL) is enough for a first test — it won't correlate
stage order, just presence, which is fine to prove the workflow itself:

```
event.dataset: "okta.system" and okta.actor.alternate_id: "j.doe@example.com"
  and okta.client.ip: "203.0.113.7"
```
Rule type: Custom query. Index pattern: `logs-okta.system-default`.
Rule action: this workflow (`Okta Credential Stuffing Response`).

For a real correlation rule (counting each stage), use an ES|QL rule that
buckets by `okta.actor.alternate_id, okta.client.ip` over a lookback window
and emits `failed_login_count`, `mfa_failure_count`,
`successful_login_count`, `post_compromise_count` as custom fields on the
alert — the workflow reads those fields if present but defaults to
`'unknown'` if they're missing, so this step can be skipped for an initial
smoke test.

## 3. Create the `case-writer` Agent Builder agent

Needs: an LLM connector, and the **Cases tools** enabled (create case, pin
observable, add comment, set status). Starter instructions:

```
You are a SOC case writer. You will be given an Okta credential-stuffing /
account-takeover alert and case requirements in the user message.

Using your Cases tools:
1. Create a case matching the title, severity, tags and settings specified.
2. Mark it in-progress.
3. Attach the alert document(s) given to you.
4. Pin the attacker IP and compromised account email as observables.
5. Add an analysis comment summarising the attack chain and evidence.

Return structured output: { "case_id": "<the case id you created>" }
```

## 4. Create the remediation script

Kibana > Security > Endpoint response actions > Scripts (or wherever your
Script Library lives) — create a script named `remediate-okta-compromise.ps1`
accepting `-SourceIP` and `-CompromisedUser` parameters. For testing, a stub
that just echoes its parameters is enough to prove the workflow's wiring:

```powershell
param(
  # Typed as [ipaddress], not [string]: `okta.client.ip` comes from the alert, and
  # the workflow only strips PowerShell metacharacters as defense in depth — this
  # type constraint is the real backstop. PowerShell rejects anything that doesn't
  # parse as an IP before the script body ever runs.
  [ipaddress]$SourceIP,
  # CompromisedUser is still [string] (usernames aren't a parseable type), so this
  # script must never pass it to Invoke-Expression, iex, or a native command line —
  # only use it as a literal string (e.g. Disable-LocalUser -Name $CompromisedUser,
  # which takes it as a value, not a string to execute).
  [string]$CompromisedUser
)
Write-Output "Would block $SourceIP at Windows Firewall"
Write-Output "Would disable local account for $CompromisedUser"
```
Copy its UUID into `consts.script_id` in `workflow.yaml`.

**Why `[ipaddress]` matters:** `okta.actor.alternate_id`/`okta.client.ip` are
attacker-influenced in this scenario — the attacker chose the usernames
(and controls the source IP) they sprayed. `workflow.yaml`'s `remediate`
step now quotes and strips PowerShell metacharacters (`"`, `` ` ``, `;`,
`|`, `$`, newlines) from both before building `scriptInput`, but that's
defense in depth at the workflow layer, not a guarantee — the script itself
rejecting a malformed `$SourceIP` via `[ipaddress]` typing is what actually
closes the hole if something still gets through.

## 5. Enroll a Windows endpoint

Fleet needs at least one **online, Windows** agent with Elastic Defend for
`find_agent`'s kuery (`status:online and
local_metadata.os.platform:windows`) to match. A Windows VM/container with
the Elastic Agent installed and the Defend integration added is enough.

## 6. Run it

Trigger the rule from step 2 (or run the workflow manually with an
equivalent `event.alerts` payload). **Expected result:** a critical-severity
case appears in Cases, in-progress, with the attacker IP and account pinned
as observables and an AI analysis comment; shortly after, a remediation
comment appears and the case closes.
