# Scenario JSON Contract

The `build-scenario` skill emits JSON in the shape below. `run-conversion-analysis` deserializes it into `engine.Scenario` dataclasses before calling `run_scenario`.

Source of truth: `engine/types.py` (dataclasses) and `engine_spec_v1.md` §2 (narrative).

## Serialization conventions

| Python type | JSON encoding | Example |
|---|---|---|
| `Decimal` | string | `"2000000"`, `"0.05"`, `"0.025"` |
| `date` | ISO 8601 string | `"1956-01-01"` |
| `Tuple[T, ...]` | array | `[{...}, {...}]` |
| `Optional[T]` | `T` or `null` | `null` |
| `dict` | object | `{"2023": "203640"}` |

Rationale: Decimals as strings avoid float rounding; dates as ISO strings are unambiguous; tuples serialize as arrays since JSON has no tuple type.

**Decimal representation.** Whole-dollar amounts and exact rates serialize without trailing zeros (`"2000000"`, not `"2000000.00"`). `scripts/serde.py::_dec()` normalizes on parse — `"2000000"` and `"2000000.00"` produce the same in-memory Decimal and the same `run_id`, but emitting the normalized form keeps JSON diff-clean. The engine's determinism layer hashes `format(Decimal, "f")`, so without the serde normalize advisors hand-typing `"2000000.00"` would produce a different `run_id` than the canonical `build_case2_scenario()`; the normalize closes that footgun.

## Full schema

The JSON below is the byte-for-byte serialization of `engine/tests/test_case2.py::build_case2_scenario()` — Case 2, the canonical validation fixture (Single, $2M IRA, FL, $640K/4yr conversion). This is the shape `build-scenario` should emit. Running this JSON through `run-conversion-analysis` produces `run_id = 9cce2a790a8b496b` (horizon_age 90, halts `BROKERAGE_INSUFFICIENT` on the full lifetime — see V1 constraints); at strict horizon 2028 the `run_id` is `598fd8a09b36c5a7` with 2025 `federal_tax.total = 36878.92`.

```json
{
  "scenario_id": "case2_aggressive",
  "tax_fact_version": "2025.11",
  "engine_version": "1.0.0",
  "allow_unverified": true,
  "client_profile": {
    "profile_version": "1.0",
    "filing_status": "Single",
    "state_code": "FL",
    "filers": [
      {
        "name": "Test Single",
        "date_of_birth": "1956-01-01",
        "enrolled_in_medicare": true,
        "rmd_applicable": true
      }
    ],
    "balances_at_start_of_first_projection_year": {
      "traditional_ira": "2000000",
      "roth_ira": "0",
      "taxable_brokerage": "400000",
      "tax_exempt_interest_annual": "0"
    },
    "annual_cash_flows": {
      "pension": [
        { "owner_index": 0, "amount": "48000", "start_year": 2025, "cola": "0" }
      ],
      "social_security": [
        { "owner_index": 0, "monthly_pia": "3200", "start_year": 2025, "cola": "0.025" }
      ],
      "wages": [],
      "qualified_dividend_yield": "0.02",
      "nonqualified_interest_yield": "0",
      "one_time_income": []
    },
    "death_events": [],
    "historical_magi": {}
  },
  "conversion_schedule": {
    "schedule_version": "1.0",
    "entries": [
      { "year": 2025, "amount": "115000" },
      { "year": 2026, "amount": "175000" },
      { "year": 2027, "amount": "175000" },
      { "year": 2028, "amount": "175000" }
    ]
  },
  "assumptions": {
    "first_projection_year": 2025,
    "horizon_mode": "age",
    "horizon_age": 90,
    "horizon_year": null,
    "growth_rate": "0.05",
    "ss_cola": "0.025",
    "pension_cola": "0",
    "bracket_inflation": "0.025",
    "irmaa_drift": "0.025",
    "conversion_tax_funding": {
      "brokerage_tax_budget": "Infinity",
      "after_budget_exhausted": "withhold",
      "baseline_tax_source": "taxable_brokerage"
    }
  }
}
```

## Field-by-field notes

### Top-level

| Field | Required | Notes |
|---|---|---|
| `scenario_id` | yes | Short slug. Stable identifier; show the advisor. |
| `tax_fact_version` | yes | Currently `"2025.11"`. Advance to `"2025.12"` if state facts change materially during Tyler review. |
| `engine_version` | yes | Currently `"1.0.0"`. |
| `allow_unverified` | yes | `true` for FL and TX always (no state income tax — the halt can't fire). For AZ/AR/OK/MS, see SKILL.md §Metadata (`RELIANCE_PILOT_MODE`). |

### `client_profile.filers[i]`

- `date_of_birth`: use year-only for age-derived DOB (`"1956-01-01"` if only age was known).
- `enrolled_in_medicare`: emit an explicit `true` or `false` based on the filer's age at `first_projection_year` (≥65 → `true`, else `false`). **Do not emit `null`** — the engine accepts it and computes identical tax math either way, but `null` fingerprints differently than `true`/`false`, so the Scenario's `run_id` drifts from canonical even when facts are identical. Override `true→false` only if the 65+ filer is explicitly deferring Medicare (rare).
- `rmd_applicable`: default `true`. Only set `false` for edge cases (inherited Roth filer, etc.).

### `client_profile.balances_at_start_of_first_projection_year`

All four fields must be strings representing non-negative Decimals. Use `"0"` for fields not present (serde normalizes `"0"` and `"0.00"` to the same in-memory Decimal, but emitting `"0"` matches canonical serialization).

### `client_profile.annual_cash_flows`

- `pension[*].owner_index`: integer index into `filers` array.
- `social_security[*].start_year`: year the filer starts claiming. Must be `≥ first_projection_year`.
- `wages[*].years`: `[start, end]` inclusive. Emit as a 2-element array of integers.
- Yields are Decimals applied annually to `balances_eoy.taxable_brokerage`. Engine handles the arithmetic.
- `one_time_income[*].kind ∈ {"ltcg", "ordinary", "exempt_interest"}`.

### `client_profile.death_events`

Optional. Omit the array (use `[]`) if no death modeled in horizon. Otherwise:
```json
[{"owner_index": 0, "year": 2030}]
```

### `client_profile.historical_magi`

JSON object. Keys are year numbers serialized as strings (JSON has no integer keys). Values are MAGI as Decimal strings. Typical keys: `first_projection_year - 2` and `- 1`. `run-conversion-analysis` will cast keys back to `int` during deserialization. Example: `{"2023": "203640.00", "2024": "208000.00"}`.

### `conversion_schedule.entries`

Each entry: `{"year": <int>, "amount": "<decimal>"}`. Engine treats missing years as `amount=0`.

### `assumptions`

- `horizon_mode`: `"age"` or `"year"`.
- `horizon_age`: required if `horizon_mode == "age"`.
- `horizon_year`: required if `horizon_mode == "year"`.
- `growth_rate`, `bracket_inflation`, `irmaa_drift`: annual rates as Decimals (e.g., 5% = `"0.05"`).
- `conversion_tax_funding.brokerage_tax_budget`: lifetime cap on brokerage-funded conversion tax. `"Infinity"` = pure Mode A (engine uses `Decimal("Infinity")`). A finite value enables Mode B.
- `conversion_tax_funding.after_budget_exhausted`: `"withhold"` (engine withholds from conversion once budget is exhausted) or `"halt"` (engine halts with `BROKERAGE_INSUFFICIENT`).
- `conversion_tax_funding.baseline_tax_source`: V1-fixed to `"taxable_brokerage"`. Do not change.

## Reference examples

Working example constructors in Python:

- `engine/tests/test_case2.py::build_case2_scenario()` — Case 2 (Single, $2M IRA, FL, $640K/4yr). The JSON schema example above is its serialized form.
- `engine/tests/test_transcripts.py::build_t2_scenario()` — T2 (MFJ 65/65 baseline).
- `engine/tests/test_transcripts.py::build_t3_scenario()` — T3 (Single age 69, $92K IRA, fill-12%).

## Minimum viable Scenario

Smallest legal Scenario JSON (for quick sanity-checking the serializer):

```json
{
  "scenario_id": "min_viable",
  "tax_fact_version": "2025.11",
  "engine_version": "1.0.0",
  "allow_unverified": true,
  "client_profile": {
    "profile_version": "1.0",
    "filing_status": "Single",
    "state_code": "FL",
    "filers": [{"name": "Min", "date_of_birth": "1956-01-01", "enrolled_in_medicare": true, "rmd_applicable": true}],
    "balances_at_start_of_first_projection_year": {
      "traditional_ira": "100000", "roth_ira": "0",
      "taxable_brokerage": "50000", "tax_exempt_interest_annual": "0"
    },
    "annual_cash_flows": {
      "pension": [], "social_security": [], "wages": [],
      "qualified_dividend_yield": "0", "nonqualified_interest_yield": "0",
      "one_time_income": []
    },
    "death_events": [],
    "historical_magi": {}
  },
  "conversion_schedule": {"schedule_version": "1.0", "entries": []},
  "assumptions": {
    "first_projection_year": 2025,
    "horizon_mode": "age",
    "horizon_age": 90,
    "horizon_year": null,
    "growth_rate": "0.05",
    "ss_cola": "0.025",
    "pension_cola": "0",
    "bracket_inflation": "0.025",
    "irmaa_drift": "0.025",
    "conversion_tax_funding": {
      "brokerage_tax_budget": "Infinity",
      "after_budget_exhausted": "withhold",
      "baseline_tax_source": "taxable_brokerage"
    }
  }
}
```

A Scenario with no conversion entries is legal — it runs as a pure baseline projection (same output the engine uses internally for `baseline_lifetime_totals`).
