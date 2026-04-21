# Rendered examples (reference only)

Actual HTML deliverables produced by the live Reliance engine, saved here
so any advisor, compliance reviewer, or stakeholder can preview what gets
produced without running the engine.

All five examples are from the same run (run_id f63befbc, John & Mary Doe,
MFJ in Texas, $1.2M trad IRA, $750K brokerage, $125K Roth, $150K/yr
conversion × 5 years starting 2028), so you can see how the same numbers
get framed across the three narrative levers plus the always-paired
disclosures and audit.

## Files

Client-facing narrative levers — advisor picks one:
- `example_lifetime_tax_saved.html` — dollar-savings framing
- `example_survivor_tax_spike.html` — surviving-spouse framing
- `example_rmd_escalation.html` — bracket-by-age chart framing

Always paired:
- `example_client_disclosures.html` — disclosures page, sent with whichever
  narrative lever the advisor picks
- `example_advisor_audit.html` — 3-page v1 audit, advisor-only

All rendered against the live hosted engine. To regenerate with different
client facts, use the full build-scenario → run-conversion-analysis →
render-advisor-report flow.
