---
name: retirement-income
version: 1.2.0
description: Plan retirement decumulation by orchestrating the public planfi MCP. Use whenever someone is at or near retirement and wants to know what order to draw down their accounts, when to claim Social Security, how to bridge health insurance before Medicare at 65, whether they have estate-tax exposure, or how to build a guaranteed bond/TIPS income floor for the first N years (sequence-of-returns protection) — e.g. "what's the tax-smart drawdown order for my taxable / traditional / Roth accounts?", "when should I claim Social Security?", "what will ACA coverage cost me until 65 if I retire early?", "will my estate owe federal estate tax?", "can I build a Treasury/TIPS ladder to floor my first 10 years of spending?".
---

# Retirement Income

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp, public, no auth).
All decumulation math — withdrawal sequencing, RMD timing, Social Security actuarial breakeven, ACA
subsidy cliffs, and estate-tax exemption logic — lives server-side. This skill only gathers inputs
and calls the tools — it does **not** compute anything locally and bakes in no defaults of its own.
Read-only.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_withdrawal_strategy`):
`analyze_withdrawal_strategy`, `optimize_social_security`, `analyze_healthcare_bridge`,
`analyze_estate_exposure`, `analyze_guaranteed_income`, `analyze_bond_ladder`, plus optional `generate_financial_plan`
(for `plan_id` chaining + a `share_url`). Use whichever name your environment exposes (bare or `mcp__planfi__`-prefixed);
below they are written bare.

If they're NOT available, tell the user to connect the MCP, then continue:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

(On claude.ai: add a custom connector pointing at https://ai.planfi.app/mcp — no auth.)

## Step 1 — (Optional but preferred) build a plan first to chain context + get a share link

> **Feed it into the forecast (not just plan_id chaining):** `generate_financial_plan` now accepts `guaranteed_income` and `bond_ladder` directly as plan inputs, so they flow into net worth, FIRE %, and Monte-Carlo backtesting — the income floor reduces the portfolio-funded spend. Use the standalone analyze tools below for a focused what-if; pass the feature into the plan to see its effect on the whole household forecast.

If the user has (or wants) a full household model, call **`generate_financial_plan`** once and
**capture the returned `plan_id`**. **All four** retirement-income tools accept `{ plan_id }` (plus
inline overrides), so they can resolve age, balances, spend, and income from the saved plan instead
of you re-sending every figure. `generate_financial_plan` also returns a **`share_url`**
(planfi.app) — none of the four decumulation tools emit a share link themselves, so this is the way
to give the user one.

This step is optional: every tool below runs from raw inputs too. Prefer the plan path when the
session already has a model, when the user is asking more than one of these questions, or when they
want a sharable artifact.

> **Engine facts to bake in:** all decimals are **fractions** (24% → `0.24`); all dollars are
> **today's (real) dollars**; brackets, exemptions, and IRMAA/ACA thresholds are approximate
> **~2026** values.

## Step 2 — Route by intent

Pick the tool that matches what the user is asking. Pass `{ plan_id }` when you have one; otherwise
pass the raw fields below.

### "What order should I draw down my accounts?" → `analyze_withdrawal_strategy`
Tax-smart withdrawal sequencing across taxable → tax-deferred → Roth, RMD-aware, with the projected
tax drag and portfolio longevity of the suggested order.
REQUIRED: `taxable_balance`, `traditional_balance`, `roth_balance`, `annual_spend`, `current_age`.
Optional: `birth_year` (RMD timing), `filing_status`, `ordinary_tax_rate`, `capital_gains_tax_rate`,
`growth_rate`, `social_security_annual`, `pension_annual`.

```
analyze_withdrawal_strategy({
  taxable_balance: 800000, traditional_balance: 1200000, roth_balance: 300000,
  annual_spend: 90000, current_age: 60
})
```

### "When should I claim Social Security?" → `optimize_social_security`
Self-orchestrating: compares claiming ages (typically 62 / FRA / 70), returns the lifetime-value
breakeven age and the recommended claim age.
Provide **either** `aime` (average indexed monthly earnings) **OR** `monthly_amount` (your estimated
benefit at FRA). Optional: `full_retirement_age`, `current_age`, `life_expectancy`, `birth_year`,
`spousal` details. Runs from sparse input — missing fields fall back to server defaults reported as
prose (see Step 3).

```
optimize_social_security({ monthly_amount: 2800, current_age: 62 })
```

### "How do I cover health insurance before Medicare?" → `analyze_healthcare_bridge`
ACA marketplace cost from the retirement age until Medicare at 65, including how premium subsidies
move with MAGI (and the subsidy-cliff interaction with the withdrawal / Roth-conversion plan).
REQUIRED: `retirement_age`, `projected_annual_income`. Optional: `household_size`, `state`,
`current_age`, `medicare_age`.

```
analyze_healthcare_bridge({ retirement_age: 58, projected_annual_income: 45000 })
```

### "Will my estate owe estate tax?" → `analyze_estate_exposure`
Self-orchestrating: projects the estate forward and compares it to the federal (and optionally
state) exemption, returning the projected taxable estate and estimated estate tax.
Useful fields: `net_worth` (or resolved via `plan_id`), `current_age`, `growth_rate`,
`life_expectancy`, `exemption_scenario` (e.g. current vs `"sunset"`), `state`. Runs from sparse
input.

```
analyze_estate_exposure({ net_worth: 14000000, current_age: 60 })
```

### "Should I take the pension lump sum or the monthly annuity? Is a SPIA/QLAC worth it?" → `analyze_guaranteed_income`
Compares a guaranteed lifetime income stream against its alternative (a pension lump sum, or a DIY
portfolio drawn at a safe withdrawal rate), returning the present-value comparison, the
breakeven/longevity age, the real income floor it buys, the after-tax monthly income, and a
recommendation. Handles both a **pension election** (lump sum vs lifetime monthly) and an **annuity
purchase** (SPIA, or a QLAC with RMD-deferral on the premium).
KEY PARAMS: `decision_type` (`'pension_election'` | `'spia'` | `'qlac'`, default `pension_election`);
`lump_sum` (the pension lump sum, or the annuity premium); `monthly_benefit` (the lifetime monthly
payout). Optional: `current_age`, `life_expectancy`, `filing_status`, `payout_start_age`, `cola`,
`discount_rate`, `expected_return` (DIY return), `desired_annual_spend` (for income-floor % of spend),
`other_taxable_income`, `tax_year`, or `plan_id` to derive age/filing/income from a saved plan.

```
analyze_guaranteed_income({
  decision_type: 'pension_election', lump_sum: 600000, monthly_benefit: 3200,
  current_age: 63, life_expectancy: 90, filing_status: 'married_joint'
})
```

### "Can I build a guaranteed income floor / bond ladder for the first N years?" → `analyze_bond_ladder`
Builds a Treasury/TIPS ladder to **floor** the first N years of retirement spending — cost today,
per-rung maturities and face values, real-vs-nominal coverage, and what % of desired spend it
guarantees. A direct complement to `analyze_withdrawal_strategy`: the ladder floors the first N
years (sequence-of-returns protection), while the withdrawal strategy handles the market-funded
remainder. Server owns all yield / inflation defaults and surfaces them via `assumed_defaults[]`.
REQUIRED: `annual_spend`, `years`. Optional: `ladder_type` (`tips` | `nominal` | `mixed`),
`real_yield`, `inflation_rate`, `existing_income_annual`, `nominal_yield`, `swr`, `filing_status`,
`state_flat_rate`, `current_age`, `start_year`, `tax_year`, or `plan_id`.

```
analyze_bond_ladder({ annual_spend: 60000, years: 10, ladder_type: 'tips' })
```

## Step 3 — Surface results honestly

For whichever tool you called:
- **Lead with the headline** — the recommended drawdown order + tax drag / longevity; the recommended
  Social Security claim age + breakeven age; the annual ACA bridge cost (net of subsidy) to age 65;
  the projected taxable estate + estimated estate tax; or — for `analyze_guaranteed_income` — the
  present-value comparison (lifetime income vs lump-sum/premium) with the net advantage, the
  breakeven age vs your longevity age, the real income floor it buys (and what % of your desired
  spend that covers), and the lump-sum-vs-lifetime / buy-annuity-vs-keep-invested recommendation.
- **Read back `disclosures.key_assumptions`** verbatim. These tools do **not** emit a structured
  `assumed_defaults[]` array — instead they apply silent Zod defaults (e.g. FRA 67, life expectancy
  ~90, growth ~5%, filing status, RMD age, exemption scenario) and expose what they assumed only as
  **prose in `disclosures.key_assumptions`**. Surface those so the user can correct any silent
  assumption.
- Honor `disclosures.not_advice` — present as planning estimates, not financial / tax advice.
- **Follow `next_actions[]`** — each is `{ tool, why, prefilled_args }` (carrying `{ plan_id }` when
  available). These commonly chain across decumulation — e.g. withdrawal → Social Security timing →
  healthcare bridge → estate exposure — and into the **tax-optimizer** skill (Roth conversions in
  gap years, multi-year tax timing). Use the server-suggested chains rather than guessing.
- **For a share link:** these four tools don't return one. If the user wants a sharable plan, run
  `generate_financial_plan` (Step 1) and surface its `share_url`.

## Recommended call sequence (typical session)

1. (preferred) `generate_financial_plan` → capture `plan_id` (+ `share_url`).
2. Route by intent → one of the four tools (with `{ plan_id }` or raw fields).
3. Read back the headline + `disclosures.key_assumptions`.
4. Follow `next_actions[]` — decumulation questions naturally chain to each other (withdrawal order
   ↔ SS timing ↔ ACA bridge ↔ estate) and into the **tax-optimizer** skill (Roth conversions in
   low-income gap years), or back to `generate_financial_plan` for a share link.

## Fictional examples

**1.** *"I'm 60 retiring with $800k taxable, $1.2M traditional, $300k Roth and need $90k/yr — what's
the tax-smart drawdown order?"*
→ `analyze_withdrawal_strategy({ taxable_balance: 800000, traditional_balance: 1200000,
roth_balance: 300000, annual_spend: 90000, current_age: 60 })`. Lead with the recommended sequence
and its tax drag / portfolio longevity; offer to chain `optimize_social_security` (when to turn on
benefits) and the **tax-optimizer** skill for Roth conversions in the pre-RMD gap years. Read back
key_assumptions (growth rate, filing status, RMD age).

**2.** *"Retiring at 58 with $45k projected income — what will ACA coverage cost me until Medicare at
65?"*
→ `analyze_healthcare_bridge({ retirement_age: 58, projected_annual_income: 45000 })`. Lead with the
net annual premium after subsidy and how it changes if MAGI rises; flag the subsidy-cliff
interaction with any Roth conversions / withdrawals, and chain `analyze_withdrawal_strategy` to keep
MAGI in the subsidy band. Read back key_assumptions (household size, state, Medicare age 65).

**3.** *"My employer is offering a $600k lump sum or $3,200/mo for life — which should I take? And
would a SPIA make more sense?"*
→ `analyze_guaranteed_income({ mode: 'pension', lump_sum: 600000, pension_monthly: 3200,
current_age: 63 })`. Lead with the PV comparison (lifetime monthly vs the lump sum) and the net
advantage, then the breakeven age vs the assumed longevity age, and the real income floor the
monthly buys. Read back the structured `assumed_defaults[]` (discount rate, DIY return, SWR, life
expectancy, survivor simplification). To compare a SPIA instead, re-run with `mode: 'annuity'`,
`premium`, and `annuity_monthly`; for a QLAC add `is_qlac: true` to see the RMD deferred on the
premium. Then chain the **tax-optimizer** skill — turning on pension income shifts your marginal
bracket and changes your Roth-conversion room in the pre-RMD gap years — and
`generate_financial_plan` for a sharable link.

*(All three examples use fictional figures — never reuse a real user's numbers in documentation.)*

## Notes

- All decimals are fractions; all dollars are today's (real) dollars; brackets / exemptions /
  IRMAA / ACA thresholds are ~2026.
- `analyze_withdrawal_strategy` (balances + spend + age) and `analyze_healthcare_bridge`
  (retirement_age + projected_annual_income) have REQUIRED inputs — ask for them before calling.
  `optimize_social_security` and `analyze_estate_exposure` self-orchestrate from sparse input.
- Pass `{ plan_id }` to reuse a saved household model; any field you also pass is a shallow override.
  All four tools accept `plan_id`.
- These four tools surface assumptions as **prose in `disclosures.key_assumptions`**, not a
  structured `assumed_defaults[]`, and they do **not** return a `share_url` — chain
  `generate_financial_plan` for a sharable link. (Server follow-up tracked in `SKILL_AUTHORING.md`.)
- **`analyze_guaranteed_income` is the exception:** unlike its four prose-only siblings, it **does**
  emit a structured `assumed_defaults[]` array (so you can read back each silent default
  individually) **and** returns a `share_url` when you pass it a `plan_id`. Surface both directly.
- Decumulation is tax-driven — the natural neighbor is the **tax-optimizer** skill (Roth conversion
  ladders, multi-year tax timing) during low-income gap years.
- Not financial or tax advice. Planning estimates only.
