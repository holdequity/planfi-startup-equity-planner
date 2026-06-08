---
name: startup-equity-planner
version: 1.0.0
description: Model the two highest-leverage pre-IPO/startup equity moves — the irreversible 30-day 83(b) early-exercise election and the §1202 QSBS gain exclusion (up to $10M legacy / $15M OBBBA, or 10× basis) — by orchestrating the public planfi MCP. Use whenever a startup founder, early employee, pre-IPO equity holder, or C-corp business seller wants to decide whether to file an 83(b) election, see the ordinary-to-LTCG conversion and holding-clock start, or estimate how much sale gain QSBS can exclude — e.g. "I'm a founder with 1,000,000 shares at a $0.001 strike, should I file an 83(b)?", "how much of my $30M exit is excluded under QSBS if I held 5 years?", "is the 83(b) worth it on 40,000 NSOs, strike $1, FMV $5, selling at $25?".
---

# Startup Equity Planner

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp, public, no auth).
All 83(b), AMT, QSBS §1202, and federal/NIIT/state tax math live server-side. This skill only
gathers inputs and calls the tools — it does **not** compute anything locally and bakes in no
thresholds, caps, or defaults of its own. Read-only.

**Related skills:** for broader RSU/ISO/NSO/ESPP grant valuation and single-stock concentration, see
**equity-comp-planner**; this skill is the pre-IPO/founder-specific companion (83(b) + QSBS).

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_startup_equity`):
`analyze_startup_equity`, `analyze_equity_compensation`, `analyze_iso_amt_crossover`, plus optional
`generate_financial_plan` (for `plan_id` chaining + a `share_url`). Use whichever name your
environment exposes (bare or `mcp__planfi__`-prefixed); below they are written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp — no auth.)

## Step 1 — (Optional) build a plan first to chain context + get a share link

If the user has (or wants) a full household model, call **`generate_financial_plan`** once and
**capture the returned `plan_id`**. `analyze_startup_equity` accepts `{ plan_id }` (plus inline
overrides), so it can resolve filing status and household context from the saved plan instead of you
re-sending every figure. `generate_financial_plan` also returns a **`share_url`** (planfi.app) — the
equity analysis tools themselves do **not** emit a share link, so this is the way to give the user one.

This step is optional: `analyze_startup_equity` runs from raw inputs too. Prefer the plan path when
the session already has a model or the user wants a sharable artifact.

> **Engine facts to bake in:** all decimals are **fractions** (24% → `0.24`); all dollars are
> **today's (real) dollars**; brackets/limits/QSBS caps are approximate **~2026** values.

## Step 2 — Route by intent

### "Should I file an 83(b)? / how much will QSBS exclude on my exit?" → `analyze_startup_equity`
The primary tool. Models the irreversible 30-day 83(b) early-exercise election (compares
83(b)-elected vs no-election lifetime tax, shows the ordinary→LTCG conversion and the holding-clock
start) AND the §1202 QSBS gain exclusion (resolves the OBBBA vs legacy regime from the acquisition
date, applies the exclusion-percent tier and the per-issuer cap). Returns the federal/NIIT/state tax
at sale and net proceeds.

REQUIRED: `shares`, `fmv_per_share` (FMV at exercise), `expected_sale_price_per_share`,
`filing_status` (`single` | `married_joint`).
Optional: `strike_price_per_share` (default 0 — founder restricted stock), `grant_type`
(`ISO` | `NSO` | `RESTRICTED`, default NSO), `make_83b_election` (default true),
`election_within_window` (default true — set false if the 30-day window is missed),
`vesting_years`, `cliff_years`, `expected_growth_rate`, `is_qsbs_eligible`, `is_qualified_ccorp`,
`gross_assets_under_ceiling`, `original_issuance`, `acquisition_date` (YYYY-MM-DD; after the
2025-07-04 OBBBA cutover → OBBBA regime, on/before → legacy), `expected_hold_years` (default 5),
`ordinary_taxable_income`, `state_flat_rate`, `state_conforms_to_qsbs`, `tax_year`, `plan_id`,
`overrides`.

```
analyze_startup_equity({
  shares: 1000000, strike_price_per_share: 0.001, fmv_per_share: 0.001,
  expected_sale_price_per_share: 30, grant_type: "RESTRICTED",
  make_83b_election: true, expected_hold_years: 5,
  is_qsbs_eligible: true, is_qualified_ccorp: true,
  gross_assets_under_ceiling: true, original_issuance: true,
  filing_status: "single"
})
```

### "When does AMT kick in if I early-exercise these ISOs?" → `analyze_iso_amt_crossover`
For ISO early-exercise, find how much bargain element you can exercise-and-hold before triggering AMT
before running `analyze_startup_equity`. Returns the `crossover_bargain_element`, `amt_at_crossover`,
and a sensitivity table. REQUIRED: `ordinary_taxable_income`. Optional: `filing_status`,
`per_share_bargain` (FMV − strike, to get a `shares_estimate`), `tax_year`, `plan_id`.

### "Value / tax all my grants (RSUs, ISOs, NSOs, ESPP)" → `analyze_equity_compensation`
For broader per-grant valuation and tax treatment across grant types (not just the 83(b)/QSBS
decision), route to `analyze_equity_compensation`. REQUIRED: `grants[]` — each
`{ type, shares, grant_price_per_share }`.

## Step 3 — Surface results honestly

For `analyze_startup_equity`:
- **Lead with the headline** — `recommended_action` (the 83(b) call + lifetime `tax_savings`) and
  `qsbs_excluded_gain`. Then the `eighty_three_b` block (`recommend`, `total_tax_with_election` vs
  `total_tax_without_election`, `ordinary_tax_at_exercise`, `amt_at_exercise`, `deadline_note`) and
  the `qsbs` block (`regime`, `exclusion_percent`, `exclusion_cap`, `excluded_gain`,
  `taxable_gain_after_exclusion`). Close with `federal_tax_at_sale`, `niit_at_sale`,
  `state_tax_at_sale`, and `net_proceeds`.
- **Read back `assumed_defaults[]`** — `{ field, assumed_value, note }` for every input the server
  defaulted (filing status, tax year, the 83(b) choice, growth rate, hold years, strike). Read these
  back so the user can correct any silent assumption.
- **Stress the irreversibility** — the `note` and `deadline_note` make clear the 83(b) election is
  irreversible and must be filed within **30 days of exercise**. Surface that prominently.
- Honor `disclosures.not_advice` (a **boolean** flag) — present results as planning estimates, not
  tax advice; surface `disclosures.key_assumptions` methodology prose when useful.
- **Follow `next_actions[]`** — each is `{ tool, why, prefilled_args }` (carrying `{ plan_id }` when
  available). `analyze_startup_equity` chains to `analyze_advanced_taxes` (run the sale-year gain
  through the full federal AMT/NIIT/bracket model). Use the server-suggested chain when present.
- **For a share link:** these tools don't return a `share_url`. If the user wants a sharable plan,
  run `generate_financial_plan` (Step 1) and surface its `share_url`.

## Recommended call sequence (typical session)

1. (optional) `generate_financial_plan` → capture `plan_id` (+ `share_url`).
2. (ISO early-exercise only) `analyze_iso_amt_crossover` → size the AMT-safe exercise.
3. `analyze_startup_equity` (with `{ plan_id }` or raw fields) → the 83(b) + QSBS decision.
4. Read back the headline + `assumed_defaults[]`; stress the 30-day irreversibility.
5. Follow `next_actions[]` → `analyze_advanced_taxes` to quantify the full federal bite.

## Fictional examples

**1.** *"I'm a founder with 1,000,000 shares at a $0.001 strike — should I file an 83(b), and how
much does QSBS exclude if I sell at $30 after 5 years?"*
→ `analyze_startup_equity({ shares: 1000000, strike_price_per_share: 0.001, fmv_per_share: 0.001,
expected_sale_price_per_share: 30, grant_type: "RESTRICTED", make_83b_election: true,
expected_hold_years: 5, is_qsbs_eligible: true, is_qualified_ccorp: true,
gross_assets_under_ceiling: true, original_issuance: true, filing_status: "single" })`. Lead with the
83(b) recommendation (near-zero exercise tax converts the whole gain to LTCG), the QSBS regime + the
excluded gain (up to the greater of the regime cap or 10× basis), and the taxable remainder + net
proceeds. Read back `assumed_defaults[]` and stress the 30-day filing window.

**2.** *"Is the 83(b) worth it on 40,000 NSOs, $1 strike, $5 FMV now, selling at $25?"*
→ `analyze_startup_equity({ shares: 40000, strike_price_per_share: 1, fmv_per_share: 5,
expected_sale_price_per_share: 25, grant_type: "NSO", make_83b_election: true, filing_status:
"single" })`. Lead with `eighty_three_b.election_savings` — electing pays ordinary tax on the small
$160k spread now and converts the $800k appreciation to LTCG, vs recognizing ordinary income at each
vest. Read back `assumed_defaults[]` (vesting schedule, growth rate, hold years).

*(Both examples use fictional figures — never reuse a real user's numbers in documentation.)*

## Notes

- All decimals are fractions; all dollars are today's (real) dollars; brackets/limits/QSBS caps are ~2026.
- `analyze_startup_equity` has REQUIRED inputs (`shares`, `fmv_per_share`,
  `expected_sale_price_per_share`, `filing_status`) — ask for them before calling. Everything else
  self-orchestrates from server defaults reported in `assumed_defaults[]`.
- The 83(b) election is **irreversible** and must be filed within **30 days of exercise** — always
  surface this caveat.
- QSBS eligibility (domestic C-corp, original issuance, gross-asset ceiling, qualifying hold) is
  modeled as flagged — tell the user to verify it with the issuer; state conformity to §1202 varies.
- Pass `{ plan_id }` to reuse a saved household model; any field you also pass is a shallow override.
- The tools return `assumed_defaults[]` (`{ field, assumed_value, note }`) for every defaulted input,
  `disclosures.key_assumptions` methodology prose, and `disclosures.not_advice` as a boolean. They do
  **not** return a `share_url` — chain `generate_financial_plan` for a sharable link.
- Not financial or tax advice. Planning estimates only.
