# Templates (reference only)

Five Jinja2 templates — the source for every deliverable this skill produces:

Client-facing narrative one-pagers (advisor picks one per client):
- `lifetime_tax_saved.html.j2` — dollar-savings headline
- `survivor_tax_spike.html.j2` — surviving-spouse reframing (MFJ only)
- `rmd_escalation.html.j2` — bracket-by-age chart

Always-paired (contract-required):
- `client_disclosures.html.j2` — "not tax advice" + methodology + assumption ledger
- `advisor_audit.html.j2` — 3-page advisor-only audit document

## Important

These files are NOT executed by the plugin. Canonical versions live on the
hosted Reliance engine and render server-side. These copies are snapshots
at plugin build time — if the server updates templates, these can drift.

Use for: design review, compliance inspection, diff-checking. Don't edit
and expect changes to show up in rendered output. Edit server-side for that.
