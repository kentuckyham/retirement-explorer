# Retirement Scenario Explorer — PRD & Technical Design

**Version:** 0.4 (post peer review)
**Last updated:** 2026-03-19
**Status:** In active development — core calculator built, Monte Carlo extension planned

### Changelog from v0.3 → v0.4

The following changes incorporate feedback from peer review:

- **Slider component lifted to module level** — fixes the anti-pattern of defining a component inside the render function, which caused React to recreate the DOM node on every render and could cause focus loss. Slider now receives `value` and `onChange` as props.
- **AI model upgraded from Haiku to Sonnet** (`claude-sonnet-4-6`) — the 2–3 paragraph analysis requires accurate numerical reasoning about bridge periods, adjusted withdrawal rates, and SS timing trade-offs. Sonnet handles this meaningfully better. Cost difference is negligible on a user-supplied key.
- **`max_tokens` raised from 450 to 650** — 450 was too tight for substantive 2–3 paragraph output.
- **Portfolio slider step changed from $50K to $25K** — allows representing values like $4.75M that previously couldn't be set.
- **Horizon cap warning added** — when the planning horizon exceeds the 40yr table limit, a visible amber warning appears in the output panel explaining the limitation rather than silently clamping.
- **"Example values" callout added** — a prominent note at the top of the inputs panel on first load signals that defaults are pre-loaded examples, not the user's numbers. Includes a "Reset to blank" affordance.
- **Dual withdrawal rate labeling improved** — the government income offset panel now uses explicit temporal labels ("During Bridge Period" vs. "After Benefits Begin") to make the two-rate concept immediately obvious.
- **Bridge period given more visual weight** — the bridge card now spans full width above the other metric cards with additional context text explaining why this is the highest-risk window.
- **AI section reframed as power-user feature** — header now reads "AI Analysis — Power User" with a clear note that this requires a paid API account.
- **Probability-Based Guardrails excluded from methods comparison** — the nested simulation required is a maintenance burden and the output isn't meaningfully different from regular Guardrails for most users.
- **Open questions resolved** — log-normal distribution confirmed, forward-looking Morningstar assumptions selected over historical, explicit Run Simulation button chosen for Monte Carlo UX.
- **End-of-year withdrawal convention documented** — the projection algorithm explicitly names the convention used to prevent confusion when users cross-check against other tools.
- **Single-person limitation strengthened** — the secondary user scope is narrowed in the spec to reflect that most retirement planning involves couples.

---

## Table of Contents

1. [Product Overview](#1-product-overview)
2. [Users & Use Cases](#2-users--use-cases)
3. [Feature Specification](#3-feature-specification)
4. [Inputs](#4-inputs)
5. [Outputs](#5-outputs)
6. [User Flows](#6-user-flows)
7. [Constraints & Non-Goals](#7-constraints--non-goals)
8. [Success Criteria](#8-success-criteria)
9. [Technical Architecture](#9-technical-architecture)
10. [Financial Algorithms](#10-financial-algorithms)
11. [Component Design](#11-component-design)
12. [State Model](#12-state-model)
13. [API Integration](#13-api-integration)
14. [Planned Extension: Monte Carlo Simulation](#14-planned-extension-monte-carlo-simulation)
15. [Known Limitations](#15-known-limitations)

---

# Part I: Product Requirements

---

## 1. Product Overview

**Retirement Scenario Explorer** is a browser-based financial planning tool that lets a user evaluate whether their retirement portfolio can sustain their desired spending across multiple time horizons, withdrawal strategies, and life scenarios.

The tool is grounded in Morningstar's *State of Retirement Income: 2025* research — specifically its safe withdrawal rate tables (Exhibit 34) and its analysis of eight dynamic withdrawal strategies (Section III). It is not a generic retirement calculator. Its distinguishing features are:

- **Horizon-aware safe withdrawal rates.** Most popular tools default to a 30-year horizon ("the 4% rule"). This tool uses Morningstar's full matrix of safe rates across horizons from 10 to 40 years and equity allocations from 0% to 100%, making it meaningfully more accurate for early retirees with 40+ year horizons.
- **Government income as a first-class input.** Social Security, CPP, and other pension income are not afterthoughts — the tool calculates and prominently displays the "adjusted rate" after government income offsets the portfolio draw, which is often the most important number for early retirees.
- **The bridge period as a central risk concept.** The gap between retirement date and government income claiming age is explicitly surfaced with prominent visual weight as the highest-risk window, informed by Morningstar's finding that 70% of retirement failures trace to the first 5 years.
- **Editable scenario slots.** Three fully user-configurable save slots allow side-by-side comparison of any scenarios the user cares about (different retirement ages, locations, spending levels, etc.) without any hardcoded narrative.

---

## 2. Users & Use Cases

### Primary user profile
Someone in the 5–10 years before planned retirement who is actively making decisions about retirement timing, spending, and strategy. Financially literate but not a professional financial planner. Likely has a complex situation: early retirement, multiple income sources, potentially international considerations.

### Secondary users
Other individuals running retirement planning numbers. Note: **this is a single-person model** — it does not support joint-life scenarios, spousal income, survivor benefits, or differing life expectancies for a couple. Most retirement planning involves two people, and joint-life modeling is where the math becomes genuinely complex. This is an explicit non-goal for v1, which meaningfully narrows the secondary user population to single individuals or one half of a couple planning independently.

### Core use cases

**UC1 — Viability check:** "Given my current portfolio and desired spending, am I safe to retire now or do I need to wait?"

**UC2 — Timing trade-off:** "How much does retiring at 51 vs. 49 change my risk profile?" Load two scenarios and compare the metrics.

**UC3 — Location/spending trade-off:** "If I move somewhere cheaper and cut spending by $40K/year, does that change my status from Aggressive to Safe?"

**UC4 — Strategy selection:** "Which of the Morningstar withdrawal strategies makes the most sense for my situation?" *(planned — requires methods comparison tab + Monte Carlo)*

**UC5 — SS claiming decision:** "How much does delaying Social Security from 67 to 70 improve my long-run picture?"

**UC6 — AI narrative (power user):** "Give me a plain-language read on this specific scenario that references my actual numbers." Requires an Anthropic API key — framed as a power-user feature, not a mainstream one.

---

## 3. Feature Specification

### F1 — Scenario save slots

Three named save slots that act as quick-load presets. Each slot stores a complete snapshot of all inputs. Users can:
- **Load** a slot by clicking it (populates all inputs instantly)
- **Save** current inputs to any slot via a "↓ Save current inputs here" button
- **Rename** any slot inline via a pencil icon
- Slots are initialized with example defaults on first load; no hardcoded semantic meaning

Slots are held in React state. They do not persist across sessions (no localStorage per artifact constraints).

### F2 — Input controls

Sliders with live value display for the five core numeric inputs. Segmented control for SS claiming age. Number inputs for SS benefit and other government income. All inputs update outputs in real time with no lag.

**Portfolio slider uses $25K steps** to allow precise values like $4.75M.

### F2a — Example values callout

On first load, a prominent callout appears at the top of the inputs panel:

> *"These are pre-loaded example values. Replace them with your own numbers, then save to a scenario slot."*

This signals to new users that the defaults are not intended to represent their situation and prevents false confidence in the initial outputs.

### F3 — Key metrics row

**Bridge period card** is displayed at full width above the other three metric cards to give it the maximum visual weight. It includes:
- The bridge duration in years (large, color-coded)
- The age range it spans (e.g., "Age 49 → 70")
- A contextual note: "No government income during this period — pure portfolio withdrawal"
- Color: red if > 15yr, amber if 10–15yr, green if < 10yr

Below the bridge card, three metric cards show:
- Your withdrawal rate (color-coded by status)
- Morningstar safe rate for this horizon and equity allocation
- Status badge (Safe / Marginal / Aggressive)

### F4 — Government income offset panel

A split view showing the benefits breakdown table (SS + CPP/pension + OAS/other) alongside a dark card showing the net portfolio draw and adjusted withdrawal rate after benefits.

**Labeling requirement (post-review):** The two different withdrawal rates must be clearly labeled with temporal context to avoid confusion:
- The metrics row rate is labeled: **"During Bridge · before benefits"**
- The adjusted rate is labeled: **"After Benefits Begin · age {ssClaimAge}+"**

This distinction is the most valuable insight the tool provides — it shows how the long-run picture transforms once government income arrives — but it will confuse a new user if the labels don't make the "when" immediately obvious.

### F5 — Portfolio projection chart

A deterministic three-line chart showing portfolio balance from retirement age to life expectancy under pessimistic (3% real), base case (5% real), and optimistic (7% real) return assumptions. Annotations: a vertical dashed line where SS kicks in (bridge end), a horizontal reference line at $0.

**Withdrawal convention: end-of-year.** The projection applies portfolio growth before subtracting the annual withdrawal (`portfolio × (1 + r) − withdrawal`). This should be noted in the chart disclaimer since beginning-of-year vs. end-of-year conventions produce different results and financially literate users may notice discrepancies against other tools.

Updated disclaimer text: *"Deterministic projections using end-of-year withdrawal convention — actual sequence of returns matters more than averages. Early losses are disproportionately harmful and not captured here."*

*Note: This chart is a candidate for replacement with a Monte Carlo fan chart in v2 — see Section 14.*

### F5a — Horizon cap warning

When the user's planning horizon (life expectancy − retirement age) exceeds 40 years, a visible amber warning appears near the safe rate metric card:

> *"Your planning horizon ({N}yr) exceeds the 40yr Morningstar table. Using 40yr rate, which may be slightly optimistic for your timeline."*

This replaces the previous behavior of silently clamping to 40 years.

### F6 — Guardrails callout

A conditional amber card that appears only when status is Marginal or Aggressive. Explains the Guardrails strategy in plain language with the specific estimated starting rate for this user.

### F7 — AI narrative analysis (power user)

**Framing:** This is explicitly a power-user feature, not a mainstream one. The section header reads "AI Analysis — Power User" and includes clear instructions for obtaining an API key.

**Model:** `claude-sonnet-4-6` — selected over Haiku because the analysis requires accurate numerical reasoning about specific financial figures. The cost difference is a few cents per call on a user-supplied key.

**Parameters:** `max_tokens: 650`, `temperature: 0.3`

The section includes:
- A labeled password field: "Your Anthropic API Key"
- Helper text with a direct link to console.anthropic.com
- A note that the key is not stored anywhere
- A clear statement that the calculator works fully without this feature

### F8 — Methods comparison *(planned, v2)*

A second tab on the right panel showing 8 of the 9 Morningstar withdrawal strategies in a comparison table, with estimated safe rates and year-1 withdrawal amounts calculated for the user's specific inputs.

**Probability-Based Guardrails (Method 7) is excluded.** The method requires nested Monte Carlo simulation (re-running a sub-simulation every year of every trial to recalculate success probability). This is a maintenance burden disproportionate to the insight gained — for most users, the output is not meaningfully different from regular Guardrails. The exclusion is noted in the UI with a brief explanation.

Methods included: Fixed Real (base case), Forgo Inflation After Loss, RMD, Guardrails (Guyton-Klinger), Actual Spending Decline, Endowment, Constant Percentage, Vanguard Floor/Ceiling.

---

## 4. Inputs

| Input | Control | Range | Step | Default |
|---|---|---|---|---|
| Portfolio Size | Slider | $1M – $10M | **$25K** | $4,800,000 |
| Annual Spending | Slider | $50K – $500K | $5K | $215,000 |
| Retirement Age | Slider | 45 – 70 | 1 year | 49 |
| Equity Allocation | Slider | 0% – 100% | 10% | 74% |
| Life Expectancy | Slider | 75 – 100 | 1 year | 92 |
| SS Claiming Age | Segmented control | 62 / 67 / 70 | — | 70 |
| SS Annual Benefit | Number input | Free | — | $53,000 |
| CPP / Pension Annual | Number input | Free | — | $15,000 |
| OAS / Other Annual | Number input | Free | — | $7,500 |

**SS benefit pre-fill behavior:** Selecting a claiming age pre-fills the SS benefit field with a configurable default (editable per-user in the `SS_BY_AGE` config constant). The field remains manually overridable.

All defaults are defined in the `INITIAL_SCENARIOS` config block at the top of the file and can be edited without touching component logic.

---

## 5. Outputs

### Derived calculations (computed on every input change)

| Output | Formula |
|---|---|
| Planning horizon | `lifeExpectancy − retirementAge` |
| Withdrawal rate | `spending / portfolio × 100` |
| Safe rate | SWR table lookup — see Section 10.1 |
| Status | `classify(withdrawalRate, safeRate)` |
| Bridge period | `ssClaimAge − retirementAge` |
| Total government benefits | `ssBenefit + cpp + oas` |
| Net portfolio draw | `max(0, spending − totalBenefits)` |
| Adjusted withdrawal rate | `netDraw / portfolio × 100` |
| Adjusted status | `classify(adjustedRate, safeRate)` |
| Bridge color | Red if > 15yr, amber if 10–15yr, green if < 10yr |
| Horizon cap warning | Show amber callout if `horizon > 40` |

### Status classification

| Label | Condition |
|---|---|
| ✓ Safe | `withdrawalRate ≤ safeRate` |
| ~ Marginal | `safeRate < withdrawalRate ≤ safeRate + 0.5%` |
| ⚠ Aggressive | `withdrawalRate > safeRate + 0.5%` |

### Chart data

Three deterministic portfolio paths computed from retirement age to life expectancy using **end-of-year withdrawal convention:**
- **Pessimistic:** 3% real return annually
- **Base Case:** 5% real return annually
- **Optimistic:** 7% real return annually

Spending inflates at 2.5%/year. After SS claiming age, government benefits (also inflated at 2.5%/year from claiming age) reduce the annual portfolio draw.

---

## 6. User Flows

### Primary flow — new user
1. Land on app → Scenario 1 loaded by default, outputs calculated
2. See "example values" callout → understand defaults need replacing
3. Adjust sliders to personal situation
4. Read metric cards — bridge period at top grabs attention first
5. Review government income offset — see how the picture changes after benefits begin
6. Rename scenario slots and save configurations for comparison
7. (Power user) Enter API key → click Analyze → read AI narrative

### Scenario comparison flow
1. Configure first scenario → click "↓ Save current inputs here" on Slot 1
2. Adjust inputs for second scenario → save to Slot 2
3. Click between Slot 1 and Slot 2 to compare metrics instantly

### SS claiming age exploration flow
1. Click "Claim at 67" → note bridge period shrinks, Year 1 SS benefit drops
2. Click "Claim at 70" → observe trade-off: longer bridge, higher lifetime benefit, better adjusted rate

---

## 7. Constraints & Non-Goals

**Constraints:**
- Single JSX file, no build step required. Must run in the Cowork artifact viewer and be copy-pasteable to CodeSandbox.
- No localStorage or sessionStorage. State is in-memory only.
- No backend. All computation is client-side. API calls go directly to the Anthropic API using a user-supplied key.
- All dependencies available via CDN (React, Recharts, Tailwind).

**Non-goals (v1):**
- Tax optimization modeling
- Asset allocation recommendations
- Medicare / healthcare cost modeling
- Inflation scenario sensitivity
- **Spouse / joint-life scenarios** — most retirement planning involves a couple, and the math for joint life expectancy, survivor benefits, spousal SS, and differing retirement ages is genuinely complex. This is the biggest limitation for general usefulness but the correct scope boundary for v1.
- Monte Carlo simulation (planned for v2 — see Section 14)
- Persistence / user accounts

---

## 8. Success Criteria

1. Load the app → three scenario slots visible, Scenario 1 loaded, example-values callout visible
2. Adjust any slider → all outputs update live with no perceptible lag
3. Click between scenario slots → inputs and outputs switch instantly
4. Rename a scenario slot inline → name persists within session
5. Save current inputs to a slot → slot subtitle reflects new values
6. Bridge period card is the first and most prominent output element
7. Government income offset panel clearly labels "during bridge" vs. "after benefits begin"
8. When horizon exceeds 40yr, an amber warning is visible near the safe rate card
9. Enter API key → click Analyze → receive 2–3 paragraph narrative referencing actual numbers from the scenario
10. The calculator section fits on a standard 1080p laptop without requiring vertical scrolling (AI section may scroll)
11. A technically literate user unfamiliar with the tool can understand the key output within 30 seconds of opening it

---

# Part II: Technical Design

---

## 9. Technical Architecture

### Stack

| Layer | Choice | Rationale |
|---|---|---|
| UI framework | React 18 (hooks) | Required for the artifact environment |
| Charts | Recharts | Available in the artifact environment; good API for financial charts |
| Styling | Inline styles | Avoids Tailwind JIT compiler dependency; artifact environment only has Tailwind's static classes |
| State | `useState`, `useMemo` | No global state needed; all derived values from a single flat inputs object |
| Computation | Client-side JS | All financial math is arithmetic; no server needed |
| AI | Anthropic Messages API | Direct browser fetch; user-supplied key |
| Build | None | Single `.jsx` file; runs directly in artifact viewer |

### File structure

```
retirement-explorer.jsx         ← entire application
retirement-explorer-prd.md      ← this document
retirement-explorer-mockup.html ← static UI mockup (reference only)
```

---

## 10. Financial Algorithms

### 10.1 Safe withdrawal rate lookup

**Source:** Morningstar *State of Retirement Income: 2025*, Exhibit 34.

The app embeds the full 11×7 table of safe withdrawal rates indexed by equity allocation (0–100% in 10% steps) and planning horizon (10, 15, 20, 25, 30, 35, 40 years) at 90% success rate, inflation-adjusted, using Morningstar's forward-looking capital market assumptions.

**Why forward-looking over historical (resolved per peer review):** Historical returns include an exceptional US equity premium era that most serious researchers don't expect to repeat. For someone deciding whether to retire today, historical parameters are arguably overconfident. Morningstar's forward-looking assumptions are more conservative and appropriate for a planning tool.

**Lookup algorithm:**
1. Round equity allocation to nearest 10% → select row
2. Clamp horizon to [10, 40] years — **if clamped, trigger UI warning (see F5a)**
3. Find the surrounding horizon band values (`lo` and `hi`)
4. Linearly interpolate: `rate = row[lo] + (h - lo) / (hi - lo) × (row[hi] - row[lo])`

```javascript
function getSafeRate(equityPct, horizon) {
  const eq = Math.max(0, Math.min(100, Math.round(equityPct / 10) * 10))
  const row = SWR[eq]
  const bands = [10, 15, 20, 25, 30, 35, 40]
  const h = Math.max(10, Math.min(40, horizon))
  const lo = bands.reduce((a, b) => (b <= h ? b : a), 10)
  const hi = bands.find(b => b >= h) ?? 40
  if (lo === hi) return row[lo]
  return +(row[lo] + ((h - lo) / (hi - lo)) * (row[hi] - row[lo])).toFixed(2)
}
```

**Embedded table (equity % → { horizon: rate }):**

```
Equity%  | 10yr | 15yr | 20yr | 25yr | 30yr | 35yr | 40yr
---------|------|------|------|------|------|------|------
100      | 8.4  | 5.8  | 4.6  | 3.8  | 3.4  | 3.2  | 3.0
90       | 8.6  | 6.0  | 4.7  | 3.9  | 3.5  | 3.2  | 3.0
80       | 8.8  | 6.1  | 4.9  | 4.1  | 3.6  | 3.3  | 3.1
70       | 9.1  | 6.3  | 5.0  | 4.2  | 3.7  | 3.4  | 3.2
60       | 9.3  | 6.5  | 5.2  | 4.3  | 3.8  | 3.4  | 3.2
50       | 9.5  | 6.6  | 5.3  | 4.4  | 3.9  | 3.5  | 3.3
40       | 9.7  | 6.7  | 5.3  | 4.4  | 3.9  | 3.5  | 3.2
30       | 9.8  | 6.8  | 5.3  | 4.4  | 3.9  | 3.5  | 3.2
20       | 9.8  | 6.8  | 5.3  | 4.4  | 3.8  | 3.4  | 3.1
10       | 9.7  | 6.7  | 5.2  | 4.3  | 3.7  | 3.3  | 3.0
0        | 9.6  | 6.5  | 5.0  | 4.1  | 3.5  | 3.0  | 2.7
```

### 10.2 Portfolio projection

**Type:** Deterministic (not stochastic). Three fixed real-return scenarios.
**Withdrawal convention:** End-of-year. Growth is applied first, then the withdrawal is subtracted. This convention matters: it is slightly more favorable to the retiree than beginning-of-year withdrawal and produces different results. Users cross-checking against other tools may notice a discrepancy if those tools use beginning-of-year.

**Algorithm:**

```
For each year y from 1 to horizon:
  age = retirementAge + y
  inflatedSpending = spending × 1.025^y
  benefitsThisYear = (age ≥ ssClaimAge)
    ? totalBenefits × 1.025^(age − ssClaimAge)
    : 0
  netWithdrawal = max(0, inflatedSpending − benefitsThisYear)
  portfolio[y] = portfolio[y-1] × (1 + realReturn) − netWithdrawal
  if portfolio[y] ≤ 0: set to 0, stop line
```

**Return assumptions:** 3% (pessimistic), 5% (base), 7% (optimistic) real annual returns.
**Inflation assumption:** 2.5%/year applied to both spending and government benefits.
**Government benefits:** Begin inflating from the year SS is claimed, not from retirement.

### 10.3 Withdrawal strategy premiums (methods comparison — planned)

**Source:** Morningstar *State of Retirement Income: 2025*, Exhibit 15 (40% equity / 30yr / 90% success).

For each dynamic withdrawal method, we have confirmed starting safe withdrawal rates at 40% equity / 30yr from the report. Probability-Based Guardrails is excluded per peer review — see rationale in F8.

| Method | 30yr Rate (40% eq) | Premium over base | Cash Flow STD | Spend/End Ratio |
|---|---|---|---|---|
| Base Case (fixed real) | 3.90% | — | 0.0% | 45/55 |
| Forgo Inflation After Loss | 4.30% | +0.40% | 5.5% | 48/52 |
| RMD Method | 4.72% | +0.82% | 43.9% | 93/7 |
| Guardrails (Guyton-Klinger) | 5.20% | +1.30% | 28.9% | 66/34 |
| Actual Spending Decline | 5.00% | +1.10% | 0.0% | 46/54 |
| Endowment (10yr rolling avg) | 5.70% | +1.80% | 38.8% | 58/42 |
| Constant Percentage of Balance | 5.70% | +1.80% | 35.0% | 58/42 |
| Vanguard Floor/Ceiling | 5.10% | +1.20% | 36.4% | 59/41 |

**Interim rate estimation (pre-Monte Carlo):**
For users with horizons other than 30 years, we apply the premium over the user's actual horizon-adjusted base case rate:

```
estimated_method_rate = getSafeRate(equityPct, horizon) + method_premium
```

**Caveat (per peer review):** This is the weakest assumption in the design. Dynamic strategies plausibly provide more relative benefit at longer horizons (more time for adjustments to compound), but the data to verify this isn't in the Morningstar report. The premium-extrapolation approach is an approximation, clearly labeled as such in the UI. This issue disappears entirely once Monte Carlo simulation is implemented, since each strategy is simulated directly at the actual horizon.

---

## 11. Component Design

### Top-level structure

```
RetirementExplorer (default export)
├── Header bar
├── Two-panel body
│   ├── LeftPanel
│   │   ├── ExampleValuesCallout
│   │   ├── SectionLabel
│   │   ├── ScenarioSlots (×3)
│   │   │   ├── Load button
│   │   │   ├── Inline name editor
│   │   │   └── Save-current button
│   │   ├── Slider (×5)            ← module-level component
│   │   ├── SSClaimingControl (segmented)
│   │   ├── SSBenefitInput
│   │   └── OtherIncomeInputs (CPP, OAS)
│   └── RightPanel
│       ├── [future: TabStrip — Analysis | Compare Methods]
│       ├── HorizonCapWarning (conditional)
│       ├── BridgePeriodCard (full-width, prominent)
│       ├── MetricCards row (×3: rate, safe rate, status)
│       ├── GovernmentBenefitsPanel
│       ├── ProjectionChart (Recharts)
│       ├── GuardrailsCallout (conditional)
│       └── AIAnalysisPanel
```

### Sub-components

**`Slider`** — **defined at module level** (not inside the main component). Receives `label`, `value`, `onChange`, `min`, `max`, `step`, `fmt` as props. This prevents React from recreating the DOM node on every render, which can cause focus loss during slider interaction.

**`MetricCard`** — dark card with label, large value, optional children and sub-text. Value color is passed as a prop.

**`Badge`** — inline status pill. Derives colors and label from the `STATUS` config object.

**`ChartTooltip`** — custom Recharts tooltip with dark styling.

**`SectionLabel`** — uppercase label used to separate input groups.

### Config constants (top of file, user-editable)

- `INITIAL_SCENARIOS` — array of 3 scenario objects defining the starting state of each save slot
- `SS_BY_AGE` — map of claiming age → pre-fill benefit amount
- `SWR` — the Morningstar Exhibit 34 table
- `STATUS` — label, color, bg, and description text for each classification

---

## 12. State Model

```javascript
// Core inputs — single flat object, all derived values computed from this
inp: {
  portfolio: number,       // $
  spending: number,        // $/yr
  retirementAge: number,   // years
  equityPct: number,       // 0–100
  lifeExpectancy: number,  // years
  ssClaimAge: number,      // 62 | 67 | 70
  ssBenefit: number,       // $/yr
  cpp: number,             // $/yr
  oas: number,             // $/yr
}

// Scenario slots — three named save states
scenarios: Array<{
  label: string,
  ...inp fields
}>

// UI state
activeSlot: number | null      // which slot is currently loaded (null if unsaved)
editingIdx: number | null      // which slot name is being inline-edited
editingVal: string             // current value of the name being edited
savedIdx: number | null        // which slot just showed "Saved!" confirmation

// AI state
apiKey: string
analysis: string
analyzing: boolean
apiError: string
```

**Derived values** (computed via `useMemo` or inline, not stored in state):

```javascript
horizon        = lifeExpectancy − retirementAge
withdrawalRate = spending / portfolio × 100
safeRate       = getSafeRate(equityPct, horizon)
bridge         = ssClaimAge − retirementAge
totalBenefits  = ssBenefit + cpp + oas
netDraw        = max(0, spending − totalBenefits)
adjustedRate   = netDraw / portfolio × 100
status         = classify(withdrawalRate, safeRate)
adjStatus      = classify(adjustedRate, safeRate)
chartData      = buildProjection(inp)   // memoized on inp
horizonCapped  = horizon > 40           // triggers UI warning
```

---

## 13. API Integration

**Endpoint:** `POST https://api.anthropic.com/v1/messages`

**Model:** `claude-sonnet-4-6` — selected over Haiku for more accurate numerical reasoning in the analysis narrative.

**Required headers:**
```
x-api-key: {user's key}
anthropic-version: 2023-06-01
anthropic-dangerous-direct-browser-access: true
content-type: application/json
```

**Request parameters:**
```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 650,
  "temperature": 0.3,
  "messages": [{ "role": "user", "content": "..." }]
}
```

**Prompt template:** Injects all current scenario values — portfolio, spending, retirement age, horizon, equity %, safe rate, actual rate, status, all government income figures, bridge period, net draw, adjusted rate. Instructs the model to write 2–3 paragraphs covering: (1) withdrawal rate safety for the specific horizon, (2) how government benefits transform the long-run picture, (3) one insight specific to this scenario that most people overlook. Explicitly prohibits generic advice, boilerplate, and "consult a financial advisor."

**Security note:** The API key is entered by the user each session, held only in React state, and never transmitted to any server other than `api.anthropic.com`. It is not logged, stored, or persisted.

---

## 14. Planned Extension: Monte Carlo Simulation

This section describes the design for a planned v2 enhancement.

### Motivation

The current deterministic projection (three fixed return lines) does not capture sequence-of-returns risk — the primary retirement failure mechanism. Additionally, the 8 dynamic withdrawal strategies are inherently path-dependent and cannot be meaningfully simulated without stochastic returns.

### Simulation engine

- 1,000 trials (sufficient for 90th-percentile accuracy; runtime < 500ms in JS)
- **Log-normal return distribution** — confirmed per peer review as the standard and appropriate choice. Fat-tailed alternatives (Student's t) would add complexity without meaningfully improving the 90th percentile success estimate this tool targets.
- Returns drawn from portfolio's blended mean and standard deviation based on equity allocation
- **Forward-looking capital market assumptions (Morningstar 2025)** — confirmed per peer review over historical parameters

**Capital market assumptions:**

| Asset class | Arithmetic mean (real) | Standard deviation |
|---|---|---|
| US/global equities | ~5.5% | ~17% |
| Fixed income | ~2.0% | ~5% |
| Correlation (stocks/bonds) | ~−0.10 | — |

*Note: Approximate real-return equivalents derived from Morningstar's nominal assumptions (page 5 of report) minus their 2.46% inflation forecast.*

### Binary search for safe starting rate

```
lo, hi = 0.01, 0.15
while (hi − lo) > 0.0001:
  mid = (lo + hi) / 2
  success = run_simulation(starting_rate=mid, ...)
  if success ≥ 0.90: lo = mid
  else: hi = mid
safe_rate = lo
```

### Strategy implementations

Each of the 7 included strategies (Probability-Based Guardrails excluded — see below) requires a `withdraw(year, portfolio, scheduled, initialRate)` function:

- **Fixed real:** `return scheduled`
- **Forgo inflation:** `return prev_year_amount if portfolio_return[year-1] < 0 else scheduled`
- **RMD:** `return portfolio / IRS_life_expectancy[age]`
- **Guardrails:** Cut 10% if current rate > 120% of initial; raise 10% if current rate < 80% of initial; skip cutbacks in final 15 years
- **Actual spending decline:** `return initial × (0.98)^year`
- **Constant percentage:** `return max(0.9 × initial, fixed_pct × portfolio)`
- **Endowment:** `return max(0.9 × initial, fixed_pct × avg(portfolio[max(0, y-10)..y]))`
- **Vanguard floor/ceiling:** `return clamp(pct × portfolio, 0.975 × prev, 1.05 × prev)`

**Excluded: Probability-Based Guardrails.** This method requires re-running a sub-simulation every year of every trial to recalculate the probability of success. This is computationally expensive (~10–50× slower), creates a significant maintenance burden, and per peer review, the output isn't meaningfully different from regular Guardrails for most users. The exclusion is documented in the UI with this explanation.

### UX: explicit Run Simulation button

**Confirmed per peer review:** The simulation triggers via an explicit "Run Simulation" button, not automatically on slider change. Rationale: slider drag + 200ms simulation + debounce = confusing intermediate states. A deliberate "Run" interaction gives the user a sense of computation happening intentionally and prevents confusion during exploratory input adjustments.

### New UI outputs enabled by Monte Carlo

1. **Live probability of success** — replaces the static Safe/Marginal/Aggressive badge with an exact percentage (e.g., "82% success over 43 years")
2. **Fan chart** — replaces the three-line deterministic chart with 5 percentile bands (10th, 25th, 50th, 75th, 90th)
3. **Methods comparison table** — one row per strategy with simulated safe rates for the user's actual horizon and equity allocation (eliminates the premium-extrapolation approximation)
4. **Spending range for dynamic strategies** — for each method, the 10th/50th/90th percentile year-by-year spending, showing the real income volatility trade-off

### Performance estimate
- 1,000 trials × 43 years × 7 strategies = 301,000 iterations
- Binary search: ~15 iterations per strategy
- Estimated total runtime: 100–500ms (acceptable for button-triggered computation)

---

## 15. Known Limitations

1. **Horizon cap at 40 years.** The SWR table ends at 40 years. Horizons exceeding 40 years use the 40yr rate with a visible UI warning. The Monte Carlo extension will remove this limitation.

2. **Deterministic projections (v1).** Does not capture sequence-of-returns risk. Addressed by Monte Carlo in v2.

3. **No inflation sensitivity.** Spending is inflated at a fixed 2.5% regardless of scenario. A high-inflation scenario input is not supported.

4. **30yr data for methods comparison (interim).** The premium-extrapolation approach for the methods comparison is an approximation. Monte Carlo eliminates this.

5. **Single-person model.** No joint life expectancy, no spousal income, no survivor benefit modeling. This is the biggest limitation for general usefulness, but the correct scope boundary for v1.

6. **Tax-agnostic.** Pre-tax vs. post-tax portfolio value is not distinguished.

7. **No session persistence.** Scenario configurations are lost on page refresh.

---

*Document prepared for implementation. All financial data sourced from Morningstar's State of Retirement Income: 2025 (published December 3, 2025). This tool is for informational and planning purposes only.*
