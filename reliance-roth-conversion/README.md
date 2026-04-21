# reliance-roth-conversion (v0.2.5)

Cowork plugin for Reliance Financial Partners advisors running
multi-year Roth conversion analyses. The deterministic tax engine
runs as a hosted service; this plugin is the advisor-facing
conversational shell.

Three skills: build-scenario (intake), run-conversion-analysis
(engine call), render-advisor-report (five deliverables).

Credentials are inline in the skill bash blocks. To rotate: change the
Fly secret and rebuild this plugin with the new value.
