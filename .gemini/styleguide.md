# Gemini Code Assist Style Guide — Reliance Roth Marketplace

This is a Claude Code plugin marketplace for the Reliance Roth Conversion advisor workflow. **Public repo.** The marketplace surfaces plugins via `.claude-plugin/marketplace.json`; each plugin lives in its own subdirectory.

There is no build step. The only "code" is JSON manifests, plugin definitions (markdown SKILL.md files, etc.), and HTML/markdown templates.

---

## Hard Rules — P0 (must fix before merge)

1. **`.claude-plugin/marketplace.json` must remain valid JSON with required fields.** Required: `name`, `plugins[]`. Each plugin entry needs `name`, `source`, `version`. Any change that breaks parsing or omits a required field is a P0.

2. **Public-repo discipline: NO secrets, credentials, API keys, internal URLs, internal client names.** This is a public marketplace; assume anyone can read this. Flag any committed `.env`, `.env.local`, API tokens, Anthropic / Google service-account keys, or proprietary client identifiers (real client names, account numbers, etc.).

3. **Plugin version monotonicity.** When a plugin's version is changed in `marketplace.json`, the corresponding plugin subdirectory must reflect the same version (e.g., in its own manifest or README). Plugin versions go forward only; a downgrade in marketplace.json without explicit reason in the PR description is a P0.

4. **Plugin source paths must resolve.** If `marketplace.json` says `"source": "./some-plugin"`, that directory MUST exist in the repo. Removing a plugin subdirectory while leaving its entry in marketplace.json (or vice versa) is a P0.

5. **No external HTTP fetches at install time.** Plugins should be self-contained; do not introduce manifests or scripts that download arbitrary code at install. Claude Code plugins are trust-boundary code — supply-chain hygiene matters.

---

## P1 — fix soon

6. **SKILL.md / agent definitions: trigger phrases must be specific.** Plugins that hijack common natural-language phrases (e.g., "open the file", "save the doc") will collide with other plugins. Triggers should be domain-specific to the plugin's use case.

7. **README and marketplace description consistency.** When `marketplace.json` plugin description changes, the plugin's own README or top-of-SKILL.md description should match. Drift makes the marketplace listing misleading.

8. **HTML deliverable templates: no inline `<script>` with credentials, no `eval()`, no remote-code fetches.** The Reliance Roth plugin renders deliverables as HTML; flag any template change that introduces JavaScript that could execute untrusted code.

---

## P2 — nice to have

9. **Markdown front-matter consistency.** SKILL.md files should consistently use YAML frontmatter (`---` delimited) with `name` and `description` keys when applicable.

10. **Plugin example/template hygiene.** When example scenarios or templates contain placeholder values, they should be obviously placeholder (`<CLIENT_NAME>`, `123 Example St`) — not realistic-looking but invented.

---

## What NOT to flag

- Markdown style nits (heading levels, line lengths) — this is a documentation-heavy repo; let it breathe.
- Generated HTML deliverables in `templates/` or `examples/` — these are reference output, not source.
- Long lists in README — plugin marketplaces tend to have lengthy descriptions; that's fine.

---

## Output format

Use the standard Gemini Code Assist review format. Group findings by severity (Critical → P0, High → P1, Medium → P2, Low → nit). Cite `file:line`. One sentence per issue + one sentence per fix.

Because this is a public repo, **err on the side of flagging anything that looks like leaked secret material, even if uncertain**. The cost of a false positive is a clarifying comment; the cost of a false negative is a public credential leak.
