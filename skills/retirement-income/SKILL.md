---
name: retirement-income
version: 1.5.0
description: Plan retirement decumulation by orchestrating the public planfi MCP. Use whenever someone is at or near retirement and wants to know what order to draw down their accounts, when to claim Social Security, how to bridge health insurance before Medicare at 65, whether they have estate-tax exposure, how to build a guaranteed bond/TIPS income floor for the first N years (sequence-of-returns protection), or how to handle long-term-care cost exposure (self-insure vs an LTC/hybrid policy, and the hit to a surviving spouse) — e.g. "what's the tax-smart drawdown order for my taxable / traditional / Roth accounts?", "when should I claim Social Security?", "what will ACA coverage cost me until 65 if I retire early?", "will my estate owe federal estate tax?", "can I build a Treasury/TIPS ladder to floor my first 10 years of spending?", "will long-term care wipe out my survivor's plan? should I self-insure or buy an LTC/hybrid policy?".
---

# Retirement Income

A thin orchestration layer over the **planfi MCP** (https://ai.planfi.app/mcp, public, no auth).
All decumulation math — withdrawal sequencing, RMD timing, Social Security actuarial breakeven, ACA
subsidy cliffs, and estate-tax exemption logic — lives server-side. This skill only gathers inputs
and calls the tools — it does **not** compute anything locally and bakes in no defaults of its own.
Each tool applies its own server-side defaults and reports them back in a structured
`assumed_defaults[]` array (read those back to the user — see Step 3). Read-only.

## Step 0 — Make sure the planfi tools are connected

This skill uses these tools (may be namespaced, e.g. `mcp__planfi__analyze_withdrawal_strategy`):
`analyze_withdrawal_strategy`, `optimize_social_security`, `analyze_healthcare_bridge`,
`analyze_estate_exposure`, `analyze_guaranteed_income`, `analyze_bond_ladder`, `analyze_cash_ladder`, `analyze_long_term_care`, `analyze_spending_strategy`, `analyze_rmd`, `analyze_irmaa`, plus optional `generate_financial_plan`
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
**capture the returned `plan_id`**. **All six** retirement-income tools accept `{ plan_id }` (plus
inline `overrides`), so they can resolve age, balances, spend, and income from the saved plan instead
of you re-sending every figure. `generate_financial_plan` returns a **`share_url`** (planfi.app), and
several of the decumulation tools also return their own `share_url` (see Step 3) — so a plan is one
of several ways to give the user a sharable link.

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
Optional: `birth_year` (RMD timing), `years` (drawdown horizon, default 30), `filing_status`
(`single` | `married_joint`), `growth_rate`, `inflation_rate`, `capital_gains_rate`, `enable_niit`,
`state_flat_rate`, `tax_year`, `plan_id`, `overrides`.

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
benefit at FRA). Optional: `fra` (Full Retirement Age, default 67), `claim_age`, `life_expectancy`
(default 90), `birth_year`, `spouse_aime` / `spouse_monthly_amount` (for spousal/survivor
coordination), `tax_year`, `plan_id`, `overrides`. Runs from sparse input — missing fields fall back
to server defaults reported in `assumed_defaults[]` (see Step 3).

```
optimize_social_security({ monthly_amount: 2800, fra: 67 })
```

### "How do I cover health insurance before Medicare?" → `analyze_healthcare_bridge`
ACA marketplace cost from the retirement age until Medicare at 65, including how premium subsidies
move with MAGI (and the subsidy-cliff interaction with the withdrawal / Roth-conversion plan).
REQUIRED: `retirement_age`, `projected_annual_income` (projected MAGI). Optional: `household_size`,
`state` (2-letter code), `spouse_age`, `current_hsa_balance`, `hsa_coverage_type`
(`individual` | `family`), `annual_hsa_contribution`, `cobra_available`, `cobra_monthly_premium`,
`cobra_months_remaining`, `annual_out_of_pocket`, `tobacco_user`, `marginal_tax_rate`,
`medicare_eligibility_age` (default 65), `tax_year`, `plan_id`, `overrides`.

```
analyze_healthcare_bridge({ retirement_age: 58, projected_annual_income: 45000 })
```

### "Will my estate owe estate tax?" → `analyze_estate_exposure`
Self-orchestrating: projects the estate forward and compares it to the federal (and optionally
state) exemption, returning the projected taxable estate and estimated estate tax.
All inputs are optional (or resolved via `plan_id`): `liquid_assets`, `real_estate_value`,
`mortgage_principal`, `current_age` (default 50), `life_expectancy` (default 90),
`federal_exemption_scenario` (`high` | `sunset`, default `sunset`), `federal_exemption_override`,
`portability`, `estimated_growth_rate` (default 0.05), `married`, `state_code` (2-letter; MA/NY/OR/WA
have tables), `tax_year`, `plan_id`, `overrides`. Runs from sparse input.

```
analyze_estate_exposure({ liquid_assets: 12000000, real_estate_value: 2000000, current_age: 60, married: true })
```

### "Should I take the pension lump sum or the monthly annuity? Is a SPIA/QLAC worth it?" → `analyze_guaranteed_income`
Compares a guaranteed lifetime income stream against its alternative (a pension lump sum, or a DIY
portfolio drawn at a safe withdrawal rate), returning the present-value comparison, the
breakeven/longevity age, the real income floor it buys, the after-tax monthly income, and a
recommendation. Handles both a **pension election** (lump sum vs lifetime monthly) and an **annuity
purchase** (SPIA, or a QLAC with RMD-deferral on the premium).
KEY PARAMS: `decision_type` (`'pension_election'` | `'spia'` | `'qlac'`, default `pension_election`);
`lump_sum` (the pension lump sum, or the annuity premium); `monthly_benefit` (the lifetime monthly
payout). Optional: `current_age` (default 65), `life_expectancy` (default 90), `filing_status`
(`single` | `married_joint`), `payout_start_age` (default 65), `cola`, `discount_rate`,
`expected_return` (DIY return), `desired_annual_spend` (for income-floor % of spend),
`survivor_pct` (joint-and-survivor fraction, e.g. 0.5), `spouse_age`, `marginal_tax_rate`,
`inflation_rate`, `other_taxable_income`, `tax_year`, `overrides`, or `plan_id` to derive
age/filing/income from a saved plan.

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

### "How do I ladder CDs / I-Bonds / EE bonds for safe near-term cash?" → `analyze_cash_ladder`
The **mass-market complement to `analyze_bond_ladder`**: instead of brokerage Treasuries/TIPS, this
ladders the FDIC-insured and savings-bond vehicles real conservative savers actually use — **CD
ladders** (brokered vs bank, early-withdrawal penalty, callable-call/reinvestment risk), **I-Bonds**
(fixed + variable composite rate, $10k annual purchase cap, federal tax-deferral + state exemption),
and **Series EE bonds** (the 20-year guaranteed double). Returns per-rung CD maturities, the blended
APY, annual maturing principal (the liquidity story), the savings-bond values at your hold horizon
(with over-cap flags), the FDIC-insured flag, and a tax block that surfaces the asymmetry — CD
interest is annually taxable and **state-taxable**, while Treasury/savings-bond interest is
**state-exempt** and federally **tax-deferred** until redemption. Use it for the near-term safe-cash
sleeve; use `analyze_bond_ladder` for the brokerage Treasury/TIPS floor.
REQUIRED: `annual_need`, `years`. Optional: `vehicle` (`cd` | `ibond` | `ee` | `mixed`), `cd_type`
(`brokered` | `bank`, default `brokered`), `cd_yield`, `callable`, `early_withdrawal_penalty_months`,
`ibond_fixed_rate`, `ibond_variable_rate`, `ee_doubling`, `annual_purchase_cap`, `inflation_rate`,
`existing_income_annual`, `filing_status`, `state_flat_rate`, `swr`, `start_year`, `current_age`,
`tax_year`, or `plan_id`.

```
analyze_cash_ladder({ annual_need: 40000, years: 5, vehicle: 'cd', cd_type: 'brokered' })
```

### "Will long-term care wipe out my plan? Should I self-insure or buy a policy?" → `analyze_long_term_care`
Models expected long-term-care cost exposure (custodial home care, assisted living, or nursing home)
and compares **self-insure vs a traditional LTC policy vs a hybrid (life/LTC) policy** on an
after-tax present-value basis — including the tail-risk note (self-insure can win the expected case
but lose at P90) and the impact on a **surviving spouse's** plan. The largest uninsured late-life
spending shock. Self-orchestrates from sparse input — server fills a care-cost table by `care_type`,
care probability, duration, and start age, all surfaced in `assumed_defaults[]`.
KEY PARAMS: `current_age`; `strategy` (`self_insure` | `traditional` | `hybrid`); `care_type`
(`custodial_home` | `assisted_living` | `nursing_home`). Optional: `care_start_age` (default 84),
`care_duration_years` (default 3), `annual_care_cost_today` (else from the `care_type` table),
`probability_need_care` (default 0.70), `life_expectancy`, `discount_rate`, `filing_status`,
`married`, `survivor_spending_factor`, a `policy` block (`annual_premium`, `premium_pay_years`,
`annual_benefit`, `benefit_period_years`, `elimination_period_days`, `inflation_protection_pct`,
`return_of_premium_or_death_benefit`), `tax_year`, `plan_id`, `overrides`.

```
analyze_long_term_care({ current_age: 60, care_type: 'assisted_living', strategy: 'self_insure', married: true })
```

### "How much should I spend each year?" → `analyze_spending_strategy`
A spending-**rate** engine that layers on top of the account-**order** question (`analyze_withdrawal_strategy`):
it decides *how much* to pull each year, not *which account* to pull it from. Compares dynamic spending
rules against a static 4% SWR baseline — **Guyton-Klinger guardrails** (raise spending in prosperity
years, cut it when the withdrawal rate drifts above a capital-preservation band), **VPW** (variable
percentage withdrawal — spend a horizon-based fraction that rises as the horizon shrinks),
the **spending smile** (Blanchett — real spending declines through the go-go years, then ticks back up
late for health costs), and a **bucket+buffer** strategy (hold N years of cash, spend it in down
markets so you never sell equities low, refill in up years). Reports the year-by-year schedule,
spend volatility vs flat 4%, the number of raises/cuts, RMD-floor years, depletion risk, and the
extra lifetime spending vs the constant-4% comparator. Tax-agnostic in v1 — pair with
`analyze_withdrawal_strategy` for the tax-smart account order and `run_backtesting` for sequence risk.
KEY PARAMS: `portfolio_balance`, `annual_spend`, `current_age`, `years`. Optional: `strategy`
(`guardrails` | `vpw` | `smile` | `bucket` | `constant_pct`), guardrail bands, raise/cut percentages,
`birth_year` (RMD floor), `filing_status`, `tax_year`, `plan_id`, `overrides`.

```
analyze_spending_strategy({ portfolio_balance: 1500000, annual_spend: 60000, current_age: 62, years: 30 })
```

### "Will my RMDs cause a tax torpedo / how much Roth-conversion runway do I have before 73 or 75?" → `analyze_rmd`
Projects required minimum distributions on the traditional balance: the SECURE-2.0 RMD start age
(73 or 75, derived from birth year), the **conversion-runway years** before RMDs begin, the first RMD
amount/divisor, the lifetime RMD total and tax, the peak-RMD year, and a **tax-torpedo** flag
(whether the first RMD pushes your marginal rate above where it would be without it) plus a
year-by-year schedule. All values are real / today's dollars. This is the *why* behind a
Roth-conversion ladder — the runway years are the low-income window to convert before RMDs stack.
REQUIRED: `traditional_balance`, `current_age`. Optional: `birth_year` (drives 73 vs 75),
`filing_status` (`single` | `married_joint`), `other_taxable_income` (for the torpedo / marginal-rate
math), `growth_rate` (REAL, default 0.05), `life_expectancy` (default 92), `tax_year`.

```
analyze_rmd({ traditional_balance: 1000000, current_age: 65, birth_year: 1961, other_taxable_income: 50000 })
```

### "What Medicare IRMAA surcharge will my income trigger / how do I stay under the next cliff?" → `analyze_irmaa`
Maps MAGI to the Medicare IRMAA tier: the annual and monthly Part B/D surcharge, the landed tier
index, the first surcharge threshold, the **next cliff** threshold with **headroom** to it, and the
step-up in surcharge if you cross it. Reflects the **2-year MAGI lookback** (the income tax year is
two years before the Medicare year) and that for `married_joint` the household MAGI covers both
spouses. Pairs with `analyze_rmd` / `analyze_roth_conversion` — RMDs and conversions both raise MAGI
into the next IRMAA cliff.
REQUIRED: `magi`. Optional: `filing_status` (`single` | `married_joint`), `current_age` (default 65),
`tax_year`.

```
analyze_irmaa({ magi: 150000, filing_status: 'single' })
```

## Step 3 — Surface results honestly

For whichever tool you called:
- **Lead with the headline** — the recommended drawdown order + tax drag / longevity; the recommended
  Social Security claim age + breakeven age; the annual ACA bridge cost (net of subsidy) to age 65;
  the projected taxable estate + estimated estate tax; or — for `analyze_guaranteed_income` — the
  present-value comparison (lifetime income vs lump-sum/premium) with the net advantage, the
  breakeven age vs your longevity age, the real income floor it buys (and what % of your desired
  spend that covers), and the lump-sum-vs-lifetime / buy-annuity-vs-keep-invested recommendation.
- **Read back `assumed_defaults[]`.** Every one of these tools returns a structured
  `assumed_defaults[]` array of `{ field, assumed_value, note }` objects naming each input the caller
  omitted and the default the server applied (e.g. FRA 67, life expectancy ~90, growth ~5%, filing
  status, RMD age, exemption scenario). Surface each one so the user can correct any assumption.
  (`disclosures.key_assumptions` is separate static prose describing the model's methodology — you
  can mention it, but the per-field defaults to confirm live in `assumed_defaults[]`.)
- Honor `disclosures.not_advice` (a boolean) — present as planning estimates, not financial / tax
  advice.
- **Follow `next_actions[]`** — each is `{ tool, why, prefilled_args }` (carrying `{ plan_id }` when
  available). Use the server-suggested chains rather than guessing. The actual edges among these
  tools: `analyze_withdrawal_strategy` → `analyze_guaranteed_income` / `analyze_bond_ladder` /
  `analyze_long_term_care` / `analyze_relocation`; `analyze_guaranteed_income` →
  `analyze_withdrawal_strategy` / `optimize_social_security` / `analyze_bond_ladder` /
  `generate_financial_plan`; `analyze_bond_ladder` → `analyze_withdrawal_strategy` /
  `optimize_social_security` / `analyze_cash_ladder` / `generate_financial_plan`;
  `analyze_cash_ladder` → `analyze_bond_ladder` / `analyze_withdrawal_strategy` (the safe-cash sleeve
  pairs with the brokerage Treasury/TIPS floor and the market-funded drawdown); `analyze_long_term_care` →
  `analyze_insurance_needs` / `analyze_survivor_stress_test` / `analyze_guaranteed_income` /
  `analyze_bond_ladder`; `analyze_rmd` → `analyze_roth_conversion` (spend the runway years) /
  `analyze_irmaa` (RMDs lift MAGI into the next surcharge cliff) / `analyze_withdrawal_strategy`;
  `analyze_irmaa` → `analyze_rmd` / `analyze_roth_conversion` (both move MAGI relative to the cliff).
  (`optimize_social_security`, `analyze_healthcare_bridge`, and
  `analyze_estate_exposure` currently emit no outgoing `next_actions` edges.) `analyze_spending_strategy`
  (the spend-*amount* question) is the natural companion to `analyze_withdrawal_strategy` (the
  account-*order* question) and to `run_backtesting` (sequence-of-returns risk on the chosen rule).
- **For a share link:** `optimize_social_security` always returns a `share_url`;
  `analyze_estate_exposure`, `analyze_withdrawal_strategy`, `analyze_healthcare_bridge`,
  `analyze_guaranteed_income`, `analyze_bond_ladder`, and `analyze_long_term_care` return a
  `share_url` only when you pass a `plan_id` that resolves a plan. Surface whichever `share_url` the
  tool returns; otherwise run `generate_financial_plan` (Step 1) for one.

## Recommended call sequence (typical session)

1. (preferred) `generate_financial_plan` → capture `plan_id` (+ `share_url`).
2. Route by intent → one of the six tools (with `{ plan_id }` or raw fields).
3. Read back the headline + each entry in `assumed_defaults[]`.
4. Follow `next_actions[]` — e.g. withdrawal order chains to a guaranteed-income / bond-ladder floor
   or a retirement relocation; guaranteed income chains to the withdrawal order, Social Security
   timing, or `generate_financial_plan`. The **tax-optimizer** skill (Roth conversions in low-income
   gap years) is a natural manual follow-on even where no server edge exists.

## Fictional examples

**1.** *"I'm 60 retiring with $800k taxable, $1.2M traditional, $300k Roth and need $90k/yr — what's
the tax-smart drawdown order?"*
→ `analyze_withdrawal_strategy({ taxable_balance: 800000, traditional_balance: 1200000,
roth_balance: 300000, annual_spend: 90000, current_age: 60 })`. Lead with the recommended sequence
and its tax drag / portfolio longevity; follow the tool's `next_actions[]` (e.g. building a
guaranteed-income / bond-ladder floor) and offer the **tax-optimizer** skill for Roth conversions in
the pre-RMD gap years. Read back each `assumed_defaults[]` entry (e.g. years 30, growth rate 5%,
filing status, RMD start age 73).

**2.** *"Retiring at 58 with $45k projected income — what will ACA coverage cost me until Medicare at
65?"*
→ `analyze_healthcare_bridge({ retirement_age: 58, projected_annual_income: 45000 })`. Lead with the
net annual premium after subsidy and how it changes if MAGI rises; flag the subsidy-cliff
interaction with any Roth conversions / withdrawals, and offer `analyze_withdrawal_strategy` to keep
MAGI in the subsidy band. Read back each `assumed_defaults[]` entry (e.g. household size 1, state CA,
marginal tax rate 24%, Medicare eligibility age 65).

**3.** *"My employer is offering a $600k lump sum or $3,200/mo for life — which should I take? And
would a SPIA make more sense?"*
→ `analyze_guaranteed_income({ decision_type: 'pension_election', lump_sum: 600000,
monthly_benefit: 3200, current_age: 63 })`. Lead with the PV comparison (lifetime monthly vs the
lump sum) and the net advantage, then the breakeven age vs the assumed longevity age, and the real
income floor the monthly buys. Read back each `assumed_defaults[]` entry (discount rate, expected
DIY return, life expectancy, payout start age). To compare a SPIA instead, re-run with
`decision_type: 'spia'`, `lump_sum` as the premium, and `monthly_benefit` as the annuity payout; for
a QLAC use `decision_type: 'qlac'` to see the RMD deferred on the premium. Follow the tool's
`next_actions[]` (withdrawal order, Social Security timing, or `generate_financial_plan` for a
sharable link), and consider the **tax-optimizer** skill — turning on pension income shifts your
marginal bracket and changes your Roth-conversion room in the pre-RMD gap years.

**4.** *"We're 60, married, and terrified long-term care will wipe out whoever's left — should we
self-insure or buy a policy?"*
→ `analyze_long_term_care({ current_age: 60, care_type: 'assisted_living', strategy: 'self_insure',
married: true })`. Lead with the expected after-tax PV cost exposure and the three-way comparison
(self-insure vs traditional vs hybrid), then the tail-risk note (self-insure may win the expected
case but lose at P90) and the surviving-spouse impact. To price a policy, re-run with
`strategy: 'traditional'` (or `'hybrid'`) and a `policy` block. Follow the tool's `next_actions[]` —
sizing life/disability coverage (`analyze_insurance_needs`), stress-testing the survivor
(`analyze_survivor_stress_test`), or pre-funding the care with a `analyze_guaranteed_income` floor or
a `analyze_bond_ladder`. Read back each `assumed_defaults[]` entry (e.g. care start age 84, duration
3 yrs, probability of needing care 0.70, the care-cost table value). Neighboring features outside
this skill: `analyze_estate_exposure` (in this skill) and `analyze_healthcare_bridge` for the
pre-Medicare gap.

**5.** *"I'm 65 with $1M in a traditional IRA and about $50k of other income — will my RMDs cause a
tax torpedo, and how many years do I have to convert before they hit?"*
→ `analyze_rmd({ traditional_balance: 1000000, current_age: 65, birth_year: 1961,
other_taxable_income: 50000 })`. Lead with the RMD start age (73 or 75 from birth year) and the
conversion-runway years before it, then the first RMD amount/divisor, lifetime RMD tax, and whether
`tax_torpedo.triggered` is true (the first RMD pushing your marginal rate up). The runway years are
the low-income window for a Roth-conversion ladder — hand off to the **tax-optimizer** skill
(`analyze_roth_conversion`) to fill that bracket, and run `analyze_irmaa` to check the RMDs don't
push MAGI over a Medicare surcharge cliff. Read back each `assumed_defaults[]` entry (growth rate,
life expectancy 92, filing status).

**6.** *"My retirement MAGI lands around $150k single — what Medicare IRMAA surcharge does that
trigger, and how much headroom before the next cliff?"*
→ `analyze_irmaa({ magi: 150000, filing_status: 'single' })`. Lead with the annual/monthly Part B/D
surcharge and the landed tier, then the next-cliff threshold and the **headroom** to it (and the
step-up if you cross it). Remind them of the **2-year lookback** (this Medicare year reflects income
two years back) and that for `married_joint` the surcharge applies to both spouses. Then check
`analyze_rmd` / the **tax-optimizer** skill — RMDs and Roth conversions both raise MAGI toward the
cliff, so plan conversions to stay just under it.

*(All six examples use fictional figures — never reuse a real user's numbers in documentation.)*

## Notes

- All decimals are fractions; all dollars are today's (real) dollars; brackets / exemptions /
  IRMAA / ACA thresholds are ~2026.
- `analyze_withdrawal_strategy` (balances + spend + age) and `analyze_healthcare_bridge`
  (retirement_age + projected_annual_income) have REQUIRED inputs — ask for them before calling.
  `optimize_social_security` and `analyze_estate_exposure` self-orchestrate from sparse input.
- Pass `{ plan_id }` to reuse a saved household model; pass `overrides` (or any inline field) to
  shallow-override on top of it. All six tools accept `plan_id` and `overrides`.
- All six tools emit a structured `assumed_defaults[]` array of `{ field, assumed_value, note }` —
  read those back to the user to confirm or correct each silent default. `disclosures.key_assumptions`
  is separate static methodology prose and `disclosures.not_advice` is a boolean.
- Share links: `optimize_social_security` always returns a `share_url`; `analyze_estate_exposure`,
  `analyze_withdrawal_strategy`, `analyze_healthcare_bridge`, `analyze_guaranteed_income`, and
  `analyze_bond_ladder` return one only when a passed `plan_id` resolves a plan. Otherwise chain
  `generate_financial_plan`.
- Decumulation is tax-driven — the natural neighbor is the **tax-optimizer** skill (Roth conversion
  ladders, multi-year tax timing) during low-income gap years.
- Two distinct decumulation questions pair up: `analyze_withdrawal_strategy` answers the account
  **order** (which account to draw, tax-smart), while `analyze_spending_strategy` answers the spend
  **amount** (Guyton-Klinger guardrails, VPW, spending smile, bucket+buffer vs a static 4% SWR). Run
  both, then `run_backtesting` to stress the chosen spending rule against historical sequence risk.
- Not financial or tax advice. Planning estimates only.
