# Reliance Roth Conversion — Claude Code plugin

A Claude Code plugin that runs multi-year Roth conversion analyses against
Reliance Financial Partners' hosted deterministic engine.

## What the plugin does

Adds three skills to Claude Code:

- **`build-scenario`** — conversational intake for a new client case. Asks
  for household, balances, income sources, conversion plan, and emits a
  validated Scenario JSON file.
- **`run-conversion-analysis`** — sends the Scenario to the hosted engine,
  saves the Run JSON, surfaces any halts in plain language.
- **`render-advisor-report`** — calls `/v1/render_report` and writes five
  HTML deliverables (three client one-pagers + Client Disclosures +
  Advisor Audit) to the client's folder.

## Prerequisites

- **Claude Code** installed. If you don't have it:
  ```bash
  curl -fsSL claude.ai/install.sh | bash
  ```
  Then make sure `~/.local/bin` is on your `PATH` (follow the installer's
  setup notes).

- **Engine credentials** — get these from Ryan:
  - `RELIANCE_ENGINE_URL` (e.g. `https://reliance-engine.fly.dev`)
  - `RELIANCE_API_KEY` (shared secret, not in this repo)

## Install

```bash
claude plugin marketplace add https://github.com/ryanjborer1/reliance-roth-marketplace
claude plugin install reliance-roth-conversion@reliance-roth-marketplace
```

## Configure engine credentials

Add the two environment variables to your shell profile so every session
picks them up.

**Bash** (`~/.bash_profile`):
```bash
echo 'export RELIANCE_ENGINE_URL="<url-from-ryan>"' >> ~/.bash_profile
echo 'export RELIANCE_API_KEY="<key-from-ryan>"' >> ~/.bash_profile
source ~/.bash_profile
```

**Zsh** (`~/.zshrc`):
```bash
echo 'export RELIANCE_ENGINE_URL="<url-from-ryan>"' >> ~/.zshrc
echo 'export RELIANCE_API_KEY="<key-from-ryan>"' >> ~/.zshrc
source ~/.zshrc
```

## Verify

```bash
claude plugin list
```

You should see `reliance-roth-conversion@reliance-roth-marketplace` listed
and enabled.

Then start a Claude Code session (`claude`) and try:

> Set up a new Roth conversion case. MFJ, FL, both age 65, $800K
> traditional IRA, $200K brokerage, $50K/yr Social Security each,
> convert $120K/yr for 4 years starting next year.

Claude should invoke the `build-scenario` skill and begin intake.

## Updating

When a new plugin version ships, pull it with:

```bash
claude plugin marketplace update reliance-roth-marketplace
```
