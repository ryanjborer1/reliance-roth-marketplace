---
name: run-conversion-analysis
description: >
  Run a prepared Scenario through the hosted Reliance engine and save
  the resulting Run JSON to the client's subfolder. Surfaces engine
  halts in plain language so the advisor can adjust inputs. Triggers
  when the advisor says "run it", "run the analysis", "execute the
  scenario", "compute the numbers", "calculate the conversion", or
  asks for the output after a Scenario was just built by
  build-scenario.
version: 0.2.6
---

# run-conversion-analysis

Execute a Scenario JSON through the hosted Reliance engine and save
the returned Run JSON. Engine v1.0.0 is numerically locked; this skill
is a thin HTTPS wrapper.

## Tone

Short confirmations. Lead with decision-relevant numbers. Never dump
per-year tables in chat. When halts happen, quote the engine's
advisor_message verbatim and add the fix in plain language.

## Inputs

Path to a Scenario JSON — usually the file just emitted by
build-scenario. If no path given, look for the most recent `*.json`
in the current client subfolder, or ask.

## How to run it

Use Bash. The engine URL and API key are inline — for a different
deployment, set `RELIANCE_ENGINE_URL` / `RELIANCE_API_KEY` env vars.

```bash
ENGINE_URL="${RELIANCE_ENGINE_URL:-https://reliance-engine.fly.dev}"
API_KEY="${RELIANCE_API_KEY:-fc82892fb8966b9350ff066a356bac6d1007aaae82456e46}"

SCENARIO_PATH="<path>"
RUN_PATH="$(dirname "$SCENARIO_PATH")/run.json"

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

## Interpreting the result

Read the Run JSON and check `status`:

- `"ok"` — `output_bundle` is present. Summarize for the advisor.
- `"halted"` — `halt` is present, no bundle. Explain in plain language.

## Advisor-facing summary (success)

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

Follow with a push-button: `Render the client one-pagers (Recommended)` /
`Try a different schedule` / `Good for now, exit`.

## Halt-code playbook

Quote `halt.advisor_message` verbatim, then add one sentence of fix:

- **STATE_FACTS_UNVERIFIED** — Pilot-grade state numbers; run anyway or wait for Tyler?
- **STATE_UNSUPPORTED** — Preview federal/IRMAA math as FL/TX while we add the state?
- **BROKERAGE_INSUFFICIENT** — Increase brokerage, shrink schedule, shorten horizon, or withhold from IRA?
- **BROKERAGE_BUDGET_EXHAUSTED** — Raise the budget or switch to withhold-after-exhaust?
- **CONVERSION_EXCEEDS_TRADITIONAL** — Trim the schedule, it exceeds the projected IRA balance.
- **FILING_STATUS_UNSUPPORTED** — V1 is MFJ/Single only; HOH/MFS aren't in yet.
- **HORIZON_BEFORE_START / DEATH_EVENT_BEFORE_START** — Fix the year in build-scenario.
- **DEATH_EVENT_BOTH_SPOUSES** — V1 models one surviving spouse at a time.
- **TAX_FACT_VERSION_NOT_LOADED / ENGINE_VERSION_MISMATCH** — V1 is 2025.11 / 1.0.0 only.
- **INTERNAL_INVARIANT** — Engine bug; capture run_id and flag.
- **NEGATIVE_INPUT** — Validation issue; fix in build-scenario.

## Determinism

`run_id` is a stable hash of Scenario + engine version + tax-fact
version. Same JSON → same run_id.

## Out of scope

- Rendering — that's render-advisor-report.
- Modifying the Scenario — route back to build-scenario.
