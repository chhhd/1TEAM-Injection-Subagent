---
name: injection-diagnostics
description: Diagnostic procedures for SQL injection, command injection, and SSTI (server-side template injection), plus the dast-harness evidence/reporting discipline (baseline+attack+control, severity vs confidence, Probe, withheld payloads) findings must satisfy in this repo. Use when a query/body parameter's value flows into a query, shell, or template engine.
---

# Injection Diagnostic Procedures

Three classes below. All three follow the same shape: **break the parser, then prove you broke the parser specifically** (not just "the app 500'd").

## Check 1 — SQL Injection

### Syntax break
Send a value containing a single unescaped quote (`'`). Compare to baseline:
- Baseline (normal value): 200, expected shape
- Attack (odd quote count): does the response change to an error? Is there a SQL-flavored error string ("syntax error", "unclosed quotation", a driver name)?

### Recovery proves causation
A raw error alone (any 500) is not evidence — plenty of apps 500 on malformed input without it meaning anything. The decisive test is **recovery**: a second payload that closes the broken syntax and comments out the rest (e.g. `' --`, `' #`, `'/*`) should return to a 200 that looks like a *valid, different* query result (not identical to baseline, not another error). Baseline (200) → attack (500, SQL error text) → control (`' --`, 200, different/expanded result) is the three-exchange pattern `dast-harness/AGENT_GUIDE.md` §5 describes.

### Boolean-based variant
If error text isn't visible (errors suppressed), try a true/false pair with identical structure: `' OR '1'='1` vs `' OR '1'='2`. A response that differs in content (more rows / fewer rows) between the two, with everything else equal, is evidence even without an explicit error message.

### What to withhold
`UNION SELECT` data extraction, stacked queries, time-based blind chains (`SLEEP()`/`WAITFOR`) — the syntax-break-and-recover pair above is sufficient to confirm; extraction payloads go in `Probe.withheld`.

## Check 2 — Command Injection

### Metacharacter test
If a parameter plausibly reaches a shell (filenames, hostnames, "ping this host" style features), try metacharacters that chain a second command without needing its output to be destructive: `; echo <marker>`, `| echo <marker>`, `` `echo <marker>` ``, `$(echo <marker>)`. Use a unique marker string per probe so you can attribute output unambiguously.

### Evidence
The marker string appearing in the response (or in an observable side effect you're explicitly authorized to check, e.g. a file the app then serves) is the evidence — not a generic timeout or 500.

### What to withhold
Anything that writes, deletes, or exfiltrates (`rm`, `curl <attacker-host>`, reverse shells). A benign echo/marker is sufficient to confirm; note stronger payloads as withheld.

## Check 3 — SSTI (Server-Side Template Injection)

### Arithmetic probe
Send a template expression appropriate to suspected engines and see if it's *evaluated* rather than reflected literally: `{{7*7}}` (Jinja2/Twig-family), `${7*7}` (various JVM template engines), `#{7*7}`. If the response contains `49` instead of the literal string you sent, the input is being evaluated as code.

### Escalate only enough to confirm engine identity
Once arithmetic evaluates, a config/object-introspection expression specific to the engine (not a shell-out) is enough to confirm which engine and that it's truly server-side template evaluation, not client-side templating reflected back. Do not escalate to RCE-class payloads (`{{ self.__init__.__globals__... }}` chains) — note the possibility and withhold the actual payload.

## Reporting discipline (dast-harness contract)

Mirrors `access-control-checklist`'s reporting section — same contract, different class:

- **Evidence is baseline + attack + control, never a single request.** For all three classes above: normal input (baseline) → payload that breaks parsing (attack) → payload that proves *recovery/evaluation*, not just breakage (control).
- **`severity` (how bad if real) and `confidence` (how sure) are separate fields.** SQLi that only proves syntax-break-and-recover without extraction is still `HIGH` severity (SQL injection is serious by class) at whatever confidence the evidence supports — don't downgrade severity because you chose not to extract data.
- **If `confidence` is below `CONFIRMED`, `rationale` must say why** (e.g. "boolean-based signal seen but not reproducible on retry → FIRM, not CONFIRMED").
- **Deliberately withheld payloads go in `Probe.withheld`, never into a lowered `confidence`.**
- **`/lookup`-style negative controls must be checked and explicitly reported as not-vulnerable**, not silently skipped — a scoring pipeline (`ground_truth.json`'s `must_not_detect`) treats a finding there as a false positive.
- **`finding_id` needs to contain the scored keyword** for the target's ground truth to credit it (e.g. `sqli-error-based-search-q`, not `query-param-issue`) — check `ground_truth.json`'s `match_any` list for the target you're testing.
