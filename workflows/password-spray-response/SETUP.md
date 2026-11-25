# Setup: Password Spray Response

This is the simplest of the six to stand up — no agents, no Fleet, no
custom index.

## 1. Create the LLM connector

Stack Management > Connectors > create a Generative AI / Anthropic
connector **named exactly** `Anthropic-Claude-Sonnet-4-6`, or edit the
`connector-id` under the `ai_analysis` step in `workflow.yaml` to match
whatever connector name you actually create.

## 2. Enable Cases

Confirm Security > Cases is enabled (default on a standard deployment).

## 3. Minimal detection rule + sample data

Any rule that produces an alert shaped like this works — the workflow only
reads `event.rule.{id,name}` and `event.alerts[]`, and its AI prompt
references (but doesn't programmatically require) `targeted_users` and an
attempt count on the alert.

A quick KQL rule + synthetic auth-log data is enough for a first test:

**Sample documents** (index into `logs-*` auth data, or your own auth log
index pattern — adjust the rule's index pattern to match):
```json
{
  "@timestamp": "2026-11-20T07:55:00.000Z",
  "event.action": "authentication_failed",
  "user.name": "jsmith",
  "source.ip": "198.51.100.23"
}
{
  "@timestamp": "2026-11-20T07:55:05.000Z",
  "event.action": "authentication_failed",
  "user.name": "mjones",
  "source.ip": "198.51.100.23"
}
{
  "@timestamp": "2026-11-20T07:55:10.000Z",
  "event.action": "authentication_failed",
  "user.name": "admin",
  "source.ip": "198.51.100.23"
}
```

**Rule:** Custom query (KQL), name it `Password Spraying Activity Detected`:
```
event.action: "authentication_failed"
```
Rule action: this workflow.

To get the richer `targeted_users`/`attempt_count` fields the AI prompt
references, use an ES|QL rule that aggregates distinct `user.name` values
and a count per `source.ip` over a short window instead, and map those
aggregates onto the alert — optional for a first smoke test, since the
prompt just says "if available".

## 4. Run it

Trigger the rule (or run manually with):
```json
{
  "event": {
    "rule": { "id": "a1b2c3d4-0000-0000-0000-000000000000", "name": "Password Spraying Activity Detected" },
    "alerts": [
      {
        "_id": "9f8e7d6c",
        "_index": ".alerts-security.alerts-default",
        "@timestamp": "2026-11-20T07:55:10.000Z",
        "targeted_users": ["jsmith", "mjones", "admin"],
        "attempt_count": 47
      }
    ]
  }
}
```

**Expected result:** a high-severity case titled `Password Spray Detected —
Password Spraying Activity Detected`, with the triggering alert attached,
an "AI Analysis" comment summarising the targeted users/attempt count, and
a final "Automated Response Complete" summary comment. No remediation
action is taken (see the note in `workflow.yaml`'s header — the
`description` field overstates what this workflow does; it's analysis-only).
