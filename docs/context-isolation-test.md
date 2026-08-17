# Context Isolation — Design & Real Test

## Why this matters for injection-agent specifically

Injection testing is the check family most likely to generate large
artifacts by construction: a length-sensitive parser bug is often probed
*with* long payloads, verbose SQL/stack-trace errors can echo attacker input
back verbatim, and boolean-based blind confirmation can take a dozen-plus
request/response pairs. If any of that lands in the orchestrator's context
directly, one injection finding can crowd out everything else the session is
tracking. Claude Code subagents get their own fresh, isolated context window
and return only a final summary — `injection-agent.md`'s "Context isolation"
section makes this an explicit design rule instead of an incidental platform
feature: process full-size artifacts internally, return only what the
`AgentFinding` contract's `MAX_EXCERPT` (2048 chars) convention actually
needs.

## Real test (not simulated)

Target: `vulnapp/app.py` (`http://127.0.0.1:5000/search?q=`) — its error
handler intentionally echoes the raw input back on a syntax error, which was
used here as a controlled way to produce a genuinely large response without
fabricating one.

**Step 1 — measure the real response sizes from the orchestrator side,
without reading the bodies into the orchestrator's own context:**

```bash
curl -s -o /dev/null -w "%{size_download} bytes" "http://127.0.0.1:5000/search?q=report"
# -> 148 bytes (baseline)

curl -s -o /dev/null -w "%{size_download} bytes" -G "http://127.0.0.1:5000/search" \
     --data-urlencode "q='AAAA...(4000 chars)"
# -> 4038 bytes (attack) — note -o /dev/null: the orchestrator learns the
#    *size* without ever pulling the body itself into its own context,
#    which is the same discipline the subagent is asked to apply one level
#    down (measure/summarize, don't transcribe)
```

**Step 2 — dispatch injection-agent (its full definition + skill) as a real
subagent against the same target, instructed to read the full 4000+
character payload/response itself and report only a capped summary.**

**Step 3 — compare what came back.**

| | Size |
|---|---|
| Raw attack response the subagent actually read | 4038 bytes |
| Raw control response the subagent actually read | 216 bytes |
| Subagent's full returned report (reached the orchestrator's context) | ≈1,550 characters total, across baseline+attack+control+impact+repro |
| Longest single verbatim excerpt in that report | 1 short fragment (`{"error":"SQL syntax error near ''AAAA...`) — explicitly not the full 4000-character payload |

The subagent's report correctly:
- Named the true full sizes (4038 / 216 bytes) as **numbers**, not as pasted content
- Included only a short excerpt sufficient to show the actual signal (the SQL-flavored error message, and the extra confidential row in the recovered response)
- Never reproduced the 4000-character filler payload anywhere in its output
- Still reached a `CONFIRMED` verdict with a correct three-exchange (baseline/attack/control) rationale — compact reporting did not cost evidentiary rigor

## What this proves

The orchestrator's context received a complete, actionable, `CONFIRMED`
finding — including accurate byte counts for artifacts it never had to
ingest itself — while the 4KB+ raw response only ever existed inside the
subagent's own isolated context. This is the concrete mechanism behind the
"large responses stay in the subagent, only summaries surface" design point:
verified with real byte counts, not asserted from the platform description
alone.
