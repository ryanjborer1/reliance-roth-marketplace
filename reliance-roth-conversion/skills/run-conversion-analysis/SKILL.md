---
name: run-conversion-analysis
description: >
  Run a prepared Scenario through the hosted Reliance engine, stash the
  Run to a temporary location, and show the advisor a chat summary so
  they can iterate on the schedule or commit to rendering. Writes
  NOTHING to the advisor's workspace until they click a render button.
  Triggers when the advisor says "run it", "run the analysis", "execute
  the scenario", "compute the numbers", or the advisor just confirmed a
  scenario in build-scenario.
version: 0.2.7
---

# run-conversion-analysis

Run the stashed Scenario through the hosted engine, stash the Run,
summarize in chat, offer a one-click commit or iterate.

## Tone

Short. Lead with decision-relevant numbers. Halts get a plain-language
fix after quoting the engine's advisor_message verbatim.

## How to run

Pull the Scenario from the ephemeral stash at
`/tmp/reliance/<client_slug>/scenario.json` (written by build-scenario).
Do NOT look in the advisor's workspace folder — that's commit-only.

```bash
ENGINE_URL="${RELIANCE_ENGINE_URL:-https://reliance-engine.fly.dev}"
API_KEY="${RELIANCE_API_KEY:-fc82892fb8966b9350ff066a356bac6d1007aaae82456e46}"

CLIENT_SLUG="<slug carried from build-scenario>"
SCENARIO_PATH="/tmp/reliance/$CLIENT_SLUG/scenario.json"
RUN_PATH="/tmp/reliance/$CLIENT_SLUG/run.json"

python3 -c "
import json, sys
scn = json.load(open(sys.argv[1]))
json.dump({'scenario': scn}, open('/tmp/_scn_envelope.json','w'))
" "$SCENARIO_PATH"

curl -sS -X POST "$ENGINE_URL/v1/run_scenario" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $API_KEY" \
  --data @/tmp/_scn_envelope.json \
  -o "$RUN_PATH" \
  -w "HTTP %{http_code}\n"
```

Read `$RUN_PATH` and check `status`:

- `"ok"` — output_bundle present. Summarize (below).
- `"halted"` — quote halt.advisor_message + fix.

## Advisor-facing summary

Numbers in chat only. No files yet.

> **Done. run_id `<first 8>`.** Projection ran `<first>` through `<last>`.
>
> **Plan vs. no plan:**
>
> | | With plan | Without |
> |---|---|---|
> | Lifetime federal tax | `<plan>` | `<baseline>` |
> | Ending Roth IRA | `<plan>` | `<baseline>` |
> | Ending traditional IRA | `<plan>` | `<baseline>` |
>
> - **Lifetime federal tax saved: `<delta>`**
> - **`<roth_delta>` shift into the Roth** — often the real story.
> - Top marginal bracket during conversions: `<bracket>`.
> - Flags: `<disclosures>` if any.

## Commit push-button

Right after the summary, open a single AskUserQuestion. Options
collapse "which scenario" + "which narrative framing" into one click.

**MFJ (4 options):**
- Render: Lifetime Tax Saved (Recommended) — dollar-savings headline
- Render: Survivor Tax Spike — reframes for widowed-spouse exposure
- Render: RMD Escalation — bracket-by-age chart
- Try a different schedule — iterate

**Single (3 options — no Survivor framing):**
- Render: Lifetime Tax Saved (Recommended)
- Render: RMD Escalation
- Try a different schedule

If "Try a different schedule": route to build-scenario with current
facts loaded, advisor changes only the conversion plan, loop repeats.

If any "Render:" option: hand to render-advisor-report with the chosen
framing. Advisor's workspace only gets written to at that point.

## Halt playbook

Quote `halt.advisor_message`, then fix:

- **STATE_FACTS_UNVERIFIED** — Pilot state; run anyway or wait for Tyler?
- **STATE_UNSUPPORTED** — FL/TX preview available while we add the state?
- **BROKERAGE_INSUFFICIENT** — Bump brokerage, shrink schedule, shorten horizon, or withhold from IRA?
- **BROKERAGE_BUDGET_EXHAUSTED** — Raise budget or withhold-after-exhaust?
- **CONVERSION_EXCEEDS_TRADITIONAL** — Trim the schedule.
- **FILING_STATUS_UNSUPPORTED** — V1 is MFJ/Single only.
- **HORIZON_BEFORE_START / DEATH_EVENT_BEFORE_START** — Fix year in build-scenario.
- **DEATH_EVENT_BOTH_SPOUSES** — V1 models one surviving spouse.
- **TAX_FACT_VERSION_NOT_LOADED / ENGINE_VERSION_MISMATCH** — V1 supports 2025.11 / 1.0.0.
- **INTERNAL_INVARIANT** — Engine bug; capture run_id.
- **NEGATIVE_INPUT** — Fix in build-scenario.

## Determinism

Same stashed Scenario → same run_id. If advisor iterates, each new
variant gets a new run_id. The conversation transcript is the history
— advisor can say "back to the $150K version" and the LLM re-runs that
variant.

## Out of scope

- Workspace writes (render-advisor-report only, commit-only).
- HTML production (same).
