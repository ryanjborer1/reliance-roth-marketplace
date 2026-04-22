# reliance-roth-conversion (v0.2.7)

Cowork plugin for Reliance Financial Partners advisors running
multi-year Roth conversion analyses. Engine runs as a hosted service on
Fly.io; this plugin is the advisor-facing conversational shell.

## Flow

1. **build-scenario** — conversational intake with push-button choices.
   Stashes Scenario to `/tmp/reliance/<client>/`. Nothing lands in the
   advisor's workspace yet.
2. **run-conversion-analysis** — runs against the hosted engine, shows
   plan-vs-baseline numbers in chat. Still nothing in the workspace.
   Offers a one-click commit (pick a narrative framing) or iterate on
   the schedule.
3. **render-advisor-report** — triggered only when the advisor commits.
   Writes three HTMLs to `<workspace>/<client_slug>/`: the chosen
   narrative one-pager (Scenario Output), the Client Disclosures page
   (Compliance Output), and the Advisor Audit (Advisor Output).

The iterative what-if loop happens entirely in chat. No JSON files, no
premature HTML generation. The advisor's workspace stays clean until
they explicitly commit.

## Reference material

`skills/render-advisor-report/templates/` and `.../examples/` bundle the
five Jinja2 source templates and five live-rendered Case examples for
compliance review and design inspection. These files are snapshots —
canonical templates render server-side.

## Credentials

Engine URL and API key are inline in the skill bash blocks. Rotate by
changing the Fly secret and pushing a new plugin version.

Advisors can override at runtime with environment variables:
- `RELIANCE_ENGINE_URL`
- `RELIANCE_API_KEY`
