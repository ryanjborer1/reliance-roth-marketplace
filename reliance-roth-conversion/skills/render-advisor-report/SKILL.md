---
name: render-advisor-report
description: >
  After the advisor clicks a Render button in run-conversion-analysis,
  render three HTML deliverables from the stashed Run and write them to
  the client's subfolder in the advisor's workspace. The three outputs
  are the chosen narrative framing (Scenario Output), the Client
  Disclosures page (Compliance Output), and the Advisor Audit (Advisor
  Output). Triggers when the advisor commits to a rendered output
  after reviewing conversion results in chat.
version: 0.2.7
---

# render-advisor-report

Render three deliverables from the currently-stashed Run and write them
to the advisor's workspace. This is the ONLY skill that writes to the
workspace folder — build-scenario and run-conversion-analysis stash to
/tmp and never touch the advisor's files.

## Inputs

- **client_slug** — carried from build-scenario / run-conversion-analysis.
- **chosen_framing** — one of: `lifetime_tax_saved`, `survivor_tax_spike`,
  `rmd_escalation`. Captured by the push-button in run-conversion-analysis.
- **report_date** — ask via push-button: current month (Recommended) /
  prior month / custom.

Do NOT ask for the other four fields the server supports (firm name,
tagline, advisor name, etc.) in V1 — defaults to Reliance branding.

## Three deliverables produced

All three land in the client's subfolder in the advisor's workspace:

- **Scenario Output** — the client-facing narrative one-pager matching
  `chosen_framing`. This is the document the advisor pairs with the
  compliance page to send to the client.
- **Compliance Output** — the Client Disclosures page ("not tax
  advice," methodology, assumption ledger). Always paired with the
  narrative one-pager when going to the client.
- **Advisor Output** — the 3-page Advisor Audit. Internal working
  document, not for client distribution.

## How to invoke

```bash
ENGINE_URL="${RELIANCE_ENGINE_URL:-https://reliance-engine.fly.dev}"
API_KEY="${RELIANCE_API_KEY:-fc82892fb8966b9350ff066a356bac6d1007aaae82456e46}"

CLIENT_SLUG="<slug carried through session>"
CHOSEN_FRAMING="<lifetime_tax_saved|survivor_tax_spike|rmd_escalation>"
REPORT_DATE="<e.g. April 2026>"
RUN_PATH="/tmp/reliance/$CLIENT_SLUG/run.json"

# Workspace output directory — create the client subfolder if it doesn't exist
# The advisor's workspace folder is whatever their current outputs_dir is.
OUT_DIR="<outputs_dir>/$CLIENT_SLUG"
mkdir -p "$OUT_DIR"

# Build request envelope
python3 -c "
import json, sys
run = json.load(open(sys.argv[1]))
body = {'run': run, 'report_date': sys.argv[2]}
json.dump(body, open('/tmp/_render_envelope.json','w'))
" "$RUN_PATH" "$REPORT_DATE"

# Call the render endpoint — returns all five HTMLs, we'll keep three
curl -sS -X POST "$ENGINE_URL/v1/render_report" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $API_KEY" \
  --data @/tmp/_render_envelope.json \
  -o /tmp/_rendered.json \
  -w "HTTP %{http_code}\n"

# Write only the three deliverables the advisor committed to
python3 -c "
import json, sys, os
data = json.load(open('/tmp/_rendered.json'))
if 'levers' not in data:
    print('RENDER FAILED:', data); sys.exit(1)
run_short = data['run_id'][:8]
slug = sys.argv[2]
outdir = sys.argv[1]
framing = sys.argv[3]

# 1. Scenario Output — the chosen narrative framing
narrative_html = data['levers'].get(framing)
if narrative_html:
    p = os.path.join(outdir, f'{slug}__{run_short}__{framing}.html')
    open(p,'w').write(narrative_html); print(p)

# 2. Compliance Output — Client Disclosures
if 'client_disclosures' in data:
    p = os.path.join(outdir, f'{slug}__{run_short}__client_disclosures.html')
    open(p,'w').write(data['client_disclosures']); print(p)

# 3. Advisor Output — Advisor Audit
if 'advisor_audit' in data:
    p = os.path.join(outdir, f'{slug}__{run_short}__advisor_audit.html')
    open(p,'w').write(data['advisor_audit']); print(p)
" "$OUT_DIR" "$CLIENT_SLUG" "$CHOSEN_FRAMING"
```

The server returns all five HTMLs on every render call. We discard the
two unchosen narrative framings — the advisor only sees the three they
committed to.

## Advisor-facing confirmation

Keep it tight. Link the three files. No other commentary:

> **Done. Three files in `<subfolder>`:**
>
> - [Scenario — `<chosen framing label>`](path)
> - [Compliance — Client Disclosures](path)
> - [Advisor — Audit](path)
>
> Pair the Scenario with the Compliance page when you send to the
> client. Keep the Advisor Audit in the file.

No trailing postamble. No "let me know if you need anything else."

## Operational notes

- **Halted runs cannot render.** Server returns 422 with
  `error: "run_halted"` + halt_code + advisor_message. Surface to the
  advisor; route back to run-conversion-analysis or build-scenario.
- **Short horizons may produce negative lifetime_federal_tax_saved** —
  mathematically honest; extend horizon or pick a different framing.
- **Engine disclosures** (e.g. `irmaa_projected`, `unverified_state_facts`)
  append server-side to the default disclosure paragraph on both the
  Scenario Output and Compliance Output.

## Filename convention

All three files use `{client_slug}__{run_short}__{kind}.html` where
`kind` is:
- `lifetime_tax_saved` | `survivor_tax_spike` | `rmd_escalation` — the
  chosen narrative framing (exactly one of these, never two)
- `client_disclosures`
- `advisor_audit`

The run_short (first 8 chars of run_id) lets the advisor tell multiple
same-client runs apart. The explicit framing name in the filename means
you can see at a glance later which story was used.

## Reference material bundled with this skill

Two reference-only folders for inspection and design review:

- `templates/` — the five Jinja2 source templates (snapshots of what
  the server renders at plugin build time).
- `examples/` — five actual rendered HTMLs from a live Doe run, so
  anyone can preview the deliverables without running the engine.

These files are NOT executed. Canonical templates render server-side.
Snapshots may drift if the server updates.

## Out of scope

- PDF export. HTML only. Browser print-to-PDF works.
- Docx export.
- Generating all three narrative framings at once. The advisor commits
  to exactly one framing per render; the other two are available only
  by re-running the skill with a different choice.
