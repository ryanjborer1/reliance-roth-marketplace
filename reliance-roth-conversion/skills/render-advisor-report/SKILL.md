---
name: render-advisor-report
description: >
  Render the five V1 deliverables from a completed Run by calling the
  hosted Reliance engine's /v1/render_report endpoint — three
  client-facing narrative one-pagers (Lifetime Tax Saved, Survivor Tax
  Spike, RMD Escalation), plus the always-included Client Disclosures
  one-pager and Advisor Audit package. Writes five HTML files to the
  client's subfolder. Triggers when the advisor says "render the
  report", "build the client one-pagers", "generate the deliverables",
  "turn this Run into a deliverable", or asks for advisor-facing
  output after a conversion analysis.
version: 0.2.5
---

# render-advisor-report

Render the five V1 deliverables from a completed Run. Rendering is
server-side; this skill POSTs the Run, receives five HTML strings, and
writes them to the client's subfolder.

## What gets produced (every time)

Three client-facing narrative framings — advisor picks one per client:

- **lifetime_tax_saved** — dollar-savings headline
- **survivor_tax_spike** — surviving-spouse reframing (MFJ only)
- **rmd_escalation** — bracket-by-age chart included

Always-paired (contract-required):

- **client_disclosures** — "not tax advice" page, methodology, key
  assumption ledger. Pair with whichever narrative lever the advisor
  chooses.
- **advisor_audit** — 3-page advisor-only working document: cover +
  client inputs, plan vs. baseline lifetime comparison + first-year
  snapshot, methodology + source ledger + known limitations. Not for
  client distribution.

## How to invoke

```bash
ENGINE_URL="${RELIANCE_ENGINE_URL:-https://reliance-engine.fly.dev}"
API_KEY="${RELIANCE_API_KEY:-fc82892fb8966b9350ff066a356bac6d1007aaae82456e46}"

RUN_PATH="<path to run.json>"
REPORT_DATE="<advisor-supplied, e.g. April 2026>"
OUT_DIR="$(dirname "$RUN_PATH")"
CLIENT_SLUG="$(basename "$OUT_DIR")"

python3 -c "
import json, sys
run = json.load(open(sys.argv[1]))
body = {'run': run, 'report_date': sys.argv[2]}
json.dump(body, open('/tmp/_render_envelope.json','w'))
" "$RUN_PATH" "$REPORT_DATE"

curl -sS -X POST "$ENGINE_URL/v1/render_report" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $API_KEY" \
  --data @/tmp/_render_envelope.json \
  -o /tmp/_rendered.json \
  -w "HTTP %{http_code}\n"

python3 -c "
import json, sys, os
data = json.load(open('/tmp/_rendered.json'))
if 'levers' not in data:
    print('RENDER FAILED:', data); sys.exit(1)
run_short = data['run_id'][:8]
slug = sys.argv[2]
outdir = sys.argv[1]
for lever, html in data['levers'].items():
    p = os.path.join(outdir, f'{slug}__{run_short}__{lever}.html')
    open(p,'w').write(html); print(p)
if 'client_disclosures' in data:
    p = os.path.join(outdir, f'{slug}__{run_short}__client_disclosures.html')
    open(p,'w').write(data['client_disclosures']); print(p)
if 'advisor_audit' in data:
    p = os.path.join(outdir, f'{slug}__{run_short}__advisor_audit.html')
    open(p,'w').write(data['advisor_audit']); print(p)
" "$OUT_DIR" "$CLIENT_SLUG"
```

Five files land in the client's subfolder, all named
`{client_slug}__{run_short}__{kind}.html`.

## Advisor-facing confirmation

Keep it tight:

> **Done. Five files saved to `<subfolder>`.**
>
> Client-facing — pick the framing that fits:
>
> - Lifetime Tax Saved — headlines the dollar savings
> - Survivor Tax Spike — reframes for widowed-spouse exposure (MFJ only)
> - RMD Escalation — includes the bracket-by-age chart
>
> Always paired:
>
> - Client Disclosures — send with whichever framing you pick
> - Advisor Audit — internal working document, not for client

## What the advisor must supply

- **report_date** — free-text month/year (e.g. "April 2026"). Ask via
  push-button with current month Recommended.

## Operational notes

- Halted runs return HTTP 422 with `error: "run_halted"` and the
  `halt_code` / `advisor_message`. Surface to advisor and prompt
  Scenario adjust.
- Short horizons may produce negative `lifetime_federal_tax_saved` —
  mathematically honest; extend horizon or pick different lever.
- Engine disclosures (`irmaa_projected`, `unverified_state_facts`)
  append server-side to the default disclosure paragraph.

## Out of scope

- PDF export. HTML only. Browser print-to-PDF works.
- Docx export. V1 is HTML only.
