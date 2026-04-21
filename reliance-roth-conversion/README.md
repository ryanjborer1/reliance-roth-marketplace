# reliance-roth-conversion (v0.2.6)

Cowork plugin for Reliance Financial Partners advisors running multi-year
Roth conversion analyses. The deterministic tax engine runs as a hosted
service on Fly.io; this plugin is the advisor-facing conversational shell.

## Three skills

- **build-scenario** — advisor-guided intake with push-button choices,
  emits a validated Scenario JSON to a client-named subfolder.
- **run-conversion-analysis** — POSTs the Scenario to the hosted engine,
  saves the Run JSON next to it.
- **render-advisor-report** — POSTs the Run to the engine's render
  endpoint, writes five HTML deliverables (three client narrative
  framings plus the always-paired Client Disclosures and Advisor Audit).

## Reference material

`skills/render-advisor-report/templates/` and `.../examples/` bundle the
five Jinja2 source templates and five live-rendered Case examples for
compliance review and design inspection. These files are snapshots — the
canonical templates render server-side.

## Credentials

Engine URL and API key are inline in the skill bash blocks. To rotate:
change the Fly secret and rebuild/push this plugin with the new value.

Advisors can override at runtime with environment variables:
- `RELIANCE_ENGINE_URL`
- `RELIANCE_API_KEY`
