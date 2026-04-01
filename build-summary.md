# Building a Retirement Scenario Explorer with Claude

## The idea

I wanted a tool to stress-test my own retirement plan — not a generic calculator that spits out one number, but something that would let me compare withdrawal strategies side by side, tweak assumptions on the fly, and actually understand the trade-offs between "safe but low" and "high but volatile" approaches to drawing down a portfolio.

The foundation: Morningstar's *State of Retirement Income: 2025* report, which evaluates eight distinct withdrawal methods across different time horizons, equity allocations, and success probabilities. I wanted to bring that research to life as an interactive tool, then go further by layering in Monte Carlo simulation.

I built the entire thing in conversation with Claude — no boilerplate generators, no starter templates. Just iterative prompting, testing in the browser, and refining.


## The stack

Deliberately minimal. The whole app is a single HTML file (~1,400 lines) with no build step:

- React 18 via CDN
- Recharts for portfolio projection charts
- Babel Standalone for in-browser JSX transpilation
- No bundler, no node_modules, no framework

This was a conscious choice. I wanted something I could open in any browser, share as a single file, and iterate on without waiting for builds. The trade-off is no TypeScript, no hot reload, no component splitting — but for a personal planning tool, that felt right.


## Phase 1: Core projection engine

The first working version had three pieces:

**Scenario presets.** Three pre-loaded retirement scenarios with different portfolio sizes, spending levels, retirement ages, and equity allocations. Switching between them instantly updates everything.

**Morningstar safe withdrawal rate lookup.** I encoded Exhibit 34 from the report — an 11×7 matrix of safe withdrawal rates by equity allocation (0–100%) and time horizon (10–40 years). The app interpolates between grid points, so a 74% equity / 43-year horizon gives a specific rate, not just "pick the nearest bucket."

**Deterministic projection chart.** Three lines showing portfolio balance over time — pessimistic, base case, and optimistic — using Morningstar's capital market assumptions (5.7% real equity return, 2.0% real bond return). The chart accounts for a "bridge period" before Social Security / CPP / OAS kicks in, with a phantom data point technique to create a sharp visual inflection at the benefit start age rather than a misleading smooth curve.


## Phase 2: Eight withdrawal methods

This was the biggest conceptual leap. The Morningstar report doesn't just evaluate the classic "4% rule" (Fixed Real) — it compares eight fundamentally different approaches to spending down a portfolio:

1. **Fixed Real** — withdraw the same inflation-adjusted dollar amount every year
2. **Forgo Inflation After Loss** — like Fixed Real, but skip the inflation raise after a down year
3. **RMD** — divide portfolio by remaining years (IRS required minimum distribution logic)
4. **Guardrails (Guyton-Klinger)** — cut 10% if withdrawal rate exceeds 120% of target; raise 10% if below 80%
5. **Actual Spending Decline** — assume real spending drops ~2%/year (the "go-go, slow-go, no-go" pattern)
6. **Endowment (10yr Avg)** — fixed percentage of the 10-year rolling average portfolio value
7. **Constant % of Balance** — fixed percentage of whatever the portfolio is worth today
8. **Vanguard Floor/Ceiling** — like Constant %, but cap annual changes at +5% / −2.5%

Each method got its own `withdraw()` function that takes a state object (current year, portfolio value, previous withdrawal, return history) and returns a dollar amount. This let both the projection chart and the Monte Carlo engine share the same logic.

The comparison table shows each method's safe starting rate, Year-1 withdrawal, income volatility, and a spend/bequest split — with color coding to flag which methods can support your target spending.

A key insight that emerged: two of the eight methods (RMD and Constant %) are "formula-driven" — they mathematically cannot deplete the portfolio. Their starting rate is meaningless in the traditional sense. The UI handles this by showing "N/A (Cannot Deplete)" instead of a rate, with color coding based on whether the Year-1 formula-based withdrawal actually meets your spending need.


## Phase 3: Monte Carlo simulation

The Morningstar estimates use a specific methodology (historical return distributions, fixed time horizon). I wanted to complement that with a from-scratch Monte Carlo engine to see how the numbers would shift under different volatility assumptions.

The engine:

- **1,000 trials** per success-rate evaluation, using a seeded PRNG (Mulberry32) for reproducibility
- **Box-Muller transform** to convert uniform random numbers into normal variates
- **Per-trial simulation** that runs each withdrawal strategy through a unique sequence of random equity and bond returns
- **Binary search** to find the maximum starting withdrawal rate that achieves 90% portfolio survival across all 1,000 trials
- Runs all 8 methods in ~250ms in the browser — no web workers needed

The results were illuminating. For example, the Monte Carlo engine found a 2.0% safe rate for Fixed Real vs. Morningstar's 3.3% estimate — the gap driven by our higher equity volatility assumption (17% vs. Morningstar's likely ~14-15%). The Guardrails method, on the other hand, showed an even larger premium over Fixed Real in simulation than in the Morningstar estimates, validating that its self-correcting mechanism genuinely works.

An **Estimated / Simulated toggle** lets you flip the comparison table between Morningstar-derived rates and Monte Carlo results, making it easy to see where the two approaches agree and where they diverge.


## Phase 4: UI polish and usability

The final round was about making the tool actually pleasant to use:

- **Strategy selector on the projection chart** — pill buttons to switch the chart between all 8 withdrawal methods, so you can see how each one affects the portfolio trajectory under pessimistic/base/optimistic deterministic scenarios
- **Info tooltips** — hover over the "i" icon next to any method for a 4-5 sentence plain-English explanation of how it works, its trade-offs, and who it's best for
- **"Fixed Real Baseline" header** on the metric cards to clarify that the summary stats (withdrawal rate, annual income, adjusted rate after benefits) refer to the base-case Fixed Real method, not whichever strategy is selected on the chart
- **Deterministic projection disclaimer** — a note explaining that the chart shows a single expected-return path per scenario, while the comparison table accounts for sequence-of-returns risk
- **1% equity slider granularity** — changed from 10% steps so you can fine-tune the allocation
- **Pre-MC consistency** — formula-driven methods show "N/A" even before running Monte Carlo, with color coding based on whether their Year-1 withdrawal meets your spending target


## Phase 5: Dark theme and public deployment

Once the functionality was solid, I wanted to share it. That meant two things: making it look professional enough for strangers to take seriously, and actually putting it on the internet.

**Dark theme overhaul.** The original light gray/white design was functional but generic. I had Claude generate a static HTML mockup first — no React, just the proposed color palette and layout — so I could evaluate the visual direction before touching the working code. Once approved, every inline style in the component got remapped: dark slate backgrounds, muted blue accents, brighter status colors tuned for dark surfaces, Inter font via Google Fonts. The trickiest part was catching all the spots where dark-on-dark text became invisible — input fields, header text, section labels. These surfaced one by one through testing.

**Landing page and custom domain.** I registered `restlessbuilder.com` through Cloudflare, set up DNS (four A records + a CNAME), and pointed it at GitHub Pages. The root site lives in a `kentuckyham.github.io` repo with a simple landing page, and the retirement explorer automatically serves from `/retirement-explorer` as a project sub-path. Total cost: ~$10/year for the domain.

**AI analysis integration.** The app already had an optional Claude API integration for scenario analysis, but I moved it from the right panel into the left sidebar so it's always accessible regardless of which tab you're on. The prompt adapts based on context: on the Scenario Analysis tab it evaluates your withdrawal rate safety, bridge period risk, and benefits offset; on the Compare Methods tab it receives all 8 strategy results (including Monte Carlo data when available) and recommends which methods best fit your specific situation. I added a simple markdown renderer so the LLM output renders with proper headings, bold text, and lists instead of raw markdown characters.

**Quality-of-life details.** A "Remember on this device" checkbox for the API key (localStorage with an opt-in toggle), auto-retry on API overload errors (up to 2 retries with backoff), and the response area only appears after you've clicked analyze — no empty placeholder wasting space.


## What I learned

**Single-file architecture scales further than you'd think.** 1,400 lines in one file sounds unwieldy, but with clear section comments and a flat component structure, it remained navigable throughout. The zero-build-step feedback loop (edit → save → refresh) kept iteration fast.

**The interesting bugs are conceptual, not syntactic.** The hardest problems weren't typos or React gotchas — they were things like "RMD shows 0% success rate" (because depleting on the final year is actually correct behavior, not failure) or "why does the chart show 6/8 methods surviving but the table says only 3/8 are safe?" (deterministic vs. stochastic framing). These required understanding the domain, not just the code.

**Withdrawal strategy choice matters more than most people realize.** The difference between Fixed Real (safest, lowest income) and Guardrails (structured flexibility, ~40% higher starting income) is enormous. Most retirement calculators only show the first one.

**Monte Carlo and published research tell different stories for a reason.** They use different assumptions, different success definitions, different return distributions. Having both side by side — with a toggle — is more honest than picking one and presenting it as truth.

**Design before implementation saves time.** Generating a static mockup of the dark theme before touching any React code meant I could evaluate (and reject or adjust) the visual direction in minutes. The alternative — restyling inline styles across 1,400 lines and then discovering you don't like the palette — would have been painful.

**Deploying is easier than you think.** GitHub Pages + a Cloudflare domain took about 20 minutes end-to-end. The single-file architecture made it trivially deployable — no build artifacts, no CI pipeline, no server. Push to main and it's live.


## What's next

The app is functional and I use it for my own planning. Possible extensions: adding tax-bracket-aware withdrawal sequencing, modeling Roth conversion ladders, or incorporating actual historical return sequences alongside the parametric Monte Carlo.

Try it at [restlessbuilder.com/retirement-explorer](https://restlessbuilder.com/retirement-explorer). Code on [GitHub](https://github.com/kentuckyham/retirement-explorer).
