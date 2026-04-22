---
name: build-scenario
description: >
  Advisor-guided intake for a multi-year Roth conversion analysis.
  Walks the advisor through the household, balances, income sources,
  and conversion plan conversationally, one thing at a time with
  push-button choices where possible. Stashes the Scenario to a
  temporary path so run-conversion-analysis can pick it up — does NOT
  save anything to the advisor's workspace folder. Triggers when the
  advisor says "build a scenario", "start a new Roth conversion case",
  "set up a conversion for <client>", "model a conversion", or invokes
  the skill cold with no context.
version: 0.2.7
---

# build-scenario

Advisor-guided intake. Produces a valid Scenario, stashes it to an
ephemeral path for the next skill, does NOT write to the advisor's
workspace folder.

## Tone

Competent, trustworthy associate — not a form validator. Avoid
developer jargon. Lead with "Let's" language. Short confirmations.

## Cold-entry opener

If the invocation has no prior context, open with exactly this shape:

> Let's set up a Roth conversion run for a client. **Most cases take
> about a minute of back-and-forth**, then I'll run the math and show
> you what the plan looks like.
>
> Here's how we'll go:
>
> - A few quick questions about the household
> - Then what they have saved
> - Then their income sources
> - Then what you're thinking for conversions
>
> **To start: what do you want to call this client?** Last name,
> nickname, deal codename — whatever you'd use in your head.

If the advisor pasted facts upfront, skip the opener; extract and ask
only for gaps.

## Flow

One thing at a time. AskUserQuestion for every discrete choice, free
text for names and numbers.

### Phase 1 — Household

1. Client name/nickname (free text).
2. Filing status (push-button): Married (MFJ) / Single.
3. Filer(s) — names + rough ages (free text).
4. State (push-button): Texas / Florida / Arizona / Arkansas. OK/MS via
   Other. Unsupported state → halt with FL/TX preview offer.

### Phase 2 — Balances

> What are the balances — traditional IRA / 401(k), any Roth, and
> taxable brokerage? Round numbers are fine.

### Phase 3 — Income

> Now income. What do they have coming in?
>
> - Social Security (monthly amount + when they claim)
> - Pensions (if any — amount and COLA)
> - Still working, or both retired?

Push-button clarifications when ambiguous (whose SS, when to claim,
high-earner estimates). Push-button for pensions (No / Mary has one /
John has one / Both). Push-button for brokerage yield (Typical /
Conservative / Dividend-heavy).

### Phase 4 — Conversion plan

Propose 3 schedule options via push-button (Moderate / Aggressive /
Conservative) grounded in Tyler's heuristics — wait for low-rate
window, fill the target bracket, before RMDs.

### Phase 5 — Summary + confirm

Compact summary under `---`: Household, Balances, Income, Conversion
plan, Defaults. Push-button: Run it (Recommended) / Adjust something.

## Defaults (apply silently, announce in summary)

| Field | Default |
|---|---|
| first_projection_year | current calendar year |
| horizon_mode | "age" |
| horizon_age | 90 |
| growth_rate | 0.05 |
| ss_cola | 0.025 |
| pension_cola | 0 |
| bracket_inflation | 0.025 |
| irmaa_drift | 0.025 |
| conversion_tax_funding | Mode A (tax from brokerage) |
| enrolled_in_medicare (65+) | true |
| enrolled_in_medicare (<65) | false |
| rmd_applicable | true |

## Output — ephemeral stash, NOT the workspace

Derive a client slug (lowercase first word of the client's name).
Create `/tmp/reliance/<client_slug>/` and write the Scenario JSON to
`scenario.json` there. This is ephemeral working state. **Do not
write to the advisor's workspace folder at this stage** — that only
happens when the advisor clicks a Render button after reviewing the
results.

Remember the client_slug in conversation state so run-conversion-analysis
and render-advisor-report can find the stash.

## Scenario JSON shape

Per `references/scenario-contract.md`. Decimals as strings.

## Metadata

- `scenario_id` — `{client_slug}_{strategy_tag}` if a label given, else
  `{YYYYMMDD}_{filingstatus}_{index}`
- `tax_fact_version` — `"2025.11"`
- `engine_version` — `"1.0.0"`
- `allow_unverified` — true silently for FL/TX (no state income tax);
  true during pilot for AZ/AR/OK/MS

## Validation

Before stashing:
- filing_status in {"MFJ", "Single"}
- len(filers) == 1 if Single else 2
- Every filer DOB implies age ≥ 18 at first_projection_year
- state_code in {"TX", "FL", "AZ", "AR", "OK", "MS"}
- Every conversion entry year ≥ first_projection_year, amount > 0

On failure, surface the specific issue and re-ask only that field.

## Out of scope

- HOH and MFS (refused).
- States outside the V1 seed list.
- Writing to the advisor's workspace folder (only render-advisor-report
  does that, only on commit).
