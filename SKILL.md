---
name: supplier-performance-review
description: >
  Analyse supplier performance against contracted SLAs in the Anaplan PRM model —
  On-Time Delivery %, Delivery Fill Rate %, Lead Time, Supplier Tier, To Review flags,
  Total Spend and AOV — and produce a ranked underperformer action list for a
  Procurement or Sourcing Manager. Use this skill whenever the user mentions suppliers,
  vendors, SLAs, OTD, DFR, fill rate, lead time, supplier tiers, supplier scorecards,
  spend at risk, late deliveries, short shipments, "who should I chase", "which
  suppliers are underperforming", supplier reviews, QBRs, dual-sourcing decisions, or
  asks about performance at a named DC, plant, location or region — even if they never
  say the words "supplier performance" or name a module. Also use it when the user asks
  a narrower follow-up about one supplier, one product or one site, because the same
  cached coordinates and the same data traps apply.
compatibility: >
  Requires the Anaplan MCP tool set and a model bound to the PRM supplier performance
  model. The fast path uses exactly two tools: get_model_context and read_cells.
---

# Supplier Performance Review

## How to be fast

The whole review is **one `read_cells` call**. Everything below exists so you can make
that call correctly on the first attempt.

Speed here is not won by thinking faster — it is won by **not making discovery calls you
don't need and not triggering retries**. A failed call is the most expensive thing that
can happen: it costs the round trip, then usually a `catalog_line_items` call to work out
why, then the retry. Three of those is the difference between 25 seconds and five
minutes.

So: copy the literals in Step 1 **exactly as written**. They are verified against the
live model. Do not "improve" the spacing, do not substitute friendlier names, and do not
call `catalog_modules` or `catalog_line_items` first to check them — that check is what
this skill replaces.

---

## The fast path

### Step 0 — Confirm binding (one call, usually instant)

```
get_model_context()
```

If `bound` is true and the model is `PRM - PR Management 1.0`, go straight to Step 1.
**Do not rebind** — rebinding burns a model-selection slot and clears the time anchor
for no benefit.

Only if nothing is bound:

```
set_model_context(
  workspace_id = "3deff6878f494b89b0a171500228b68d",
  model_id     = "F23AD4B240BC44709046726A36FB6F9F"
)
```

If `catalog_models` shows several PRM variants, show the list and let the user pick —
tier thresholds and SLA baselines differ between model copies, so guessing produces a
confident wrong answer.

### Step 1 — The one read

Module: **`KPI220 - Supplier Performance Reporting`** (entity id `102000000594`).
It has three dimensions and no others:

| Dimension (exact string) | Use |
|---|---|
| `HLS080 - Supplier Leaf #` | leaf |
| `HLP080 - Product Leaf #` | leaf |
| `HLL080 - Location Leaf #` | slice to the user's site |

```
read_cells(
  module = "KPI220 - Supplier Performance Reporting",
  line_items = [
    "Final OTD%", "Final DFR%", "Final Lead Time",
    "OTD Variance", "DFR Variance", "Lead Time Variance",
    "Final Supplier Tier", "To Review?",
    "Total Spend", "AOV"
  ],
  slice = { "HLL080 - Location Leaf #": "Florida DC" },
  leaf_dimensions = ["HLS080 - Supplier Leaf #", "HLP080 - Product Leaf #"],
  limit = 99
)
```

At a single DC this returns the complete picture — around 42 rows for Florida DC, six
suppliers across seven products — in one call, well inside 30 seconds.

**`limit = 99`, not 100 and not 200.** The read is an exclusive-read SQL query capped at
100 rows, and the tool adds one row internally to test for a next page. Any limit above
99 fails with `CORE_API_HTTP_ERROR 400` before touching data. This was the single most
common cause of a slow run: the 200 looked harmless, failed instantly, and sent the whole
session into exploratory debugging.

**Exact line item spellings.** The percentage items have **no space before the `%`** —
`Final OTD%`, not `Final OTD %`. One wrong character fails the call. Contract baselines
are `Contract OTD%`, `Contract DFR%`, `Contract Lead Time` — the model does not use the
word "SLA" anywhere in a line item name. They are omitted from the call above on purpose:
the variance items already encode actual-minus-contract, so pulling the baselines adds
payload without adding an answer. Add them only if the user asks what was contracted.

Optional extras, each costing a little payload — add only on demand:
`Priority (Compound Alerts)`, `Prior OTD`, `Prior DFR`, `Prior Supplier Tier`,
`Change in Supplier Tier?`, `AOV as % of Spend`, `PY Total Spend`, `Override OTD%`.

### If the read still fails

Work in this order and stop as soon as it succeeds — do not escalate to `sql_schema`
exploration, which is what turns a 30-second answer into a five-minute one:

1. **Row-cap error** — drop `limit` to 99. Nothing else.
2. **Unknown line item** — remove the optional items, keep the ten core ones, retry.
3. **Unaccounted dimension** — the error names it. Add it to `slice`, retry.
4. **Still failing after two attempts** — now, and only now, run
   `catalog_line_items(module_id="102000000594", include_dimensions=false, limit=100)`
   to re-sync the names. The model version has probably changed, so tell the user the
   cached coordinates in this skill need updating.

### Step 2 — Scope

If the user named a location, use it. If they named nothing, **do not stop to ask** — run
the site they are known to manage, or the top-level roll-up, state the scope in the first
line of your answer, and offer to narrow. A clarifying question costs a full round trip;
a stated assumption costs one sentence.

Interrupt only when a name is genuinely ambiguous between two real options — for example
a name that matches both a region and a DC inside it.

---

## Reading the values correctly

**The percentage items and the variance items are stored in different units.** This trips
up every first reading and silently produces nonsense.

| Item | Stored as | Means |
|---|---|---|
| `Final OTD%`, `Final DFR%` | ratio — `0.625` | 62.5% — multiply by 100 |
| `OTD Variance`, `DFR Variance` | **percentage points** — `-17.5` | 17.5 points below contract |
| `Lead Time Variance`, `Final Lead Time` | days — `-0.75` | 0.75 days faster than contract |

So a line showing `final otd% = 0.625` with `otd variance = -17.5` is running at 62.5%
OTD against an 80% contract. Do not multiply the variance by 100, and do not report the
ratio as if it were already a percentage.

Expect long floats — `-30.000000000000004`, `19.999999999999996`. These are float
artefacts of ±30 and ±20. Round for presentation and treat anything within 1e-9 of zero
as exactly zero, because the zero test is what separates real data from missing data.

---

## Known traps

Each of these has produced a wrong answer before. They are properties of this module, not
general advice.

**1. Ratio measures do not roll up.**
`Final OTD%`, `Final DFR%` and all three variances come back **blank** at the
`All Products` / `All Suppliers` roll-up — they only populate at leaf grain. Blank at
roll-up means the measure doesn't aggregate, *not* that suppliers are failing.
`Total Spend` does roll up correctly, so use it to validate your leaf sums. **All
performance analysis happens at Supplier × Product leaf.**

**2. `0% / 0%` with zero variance is missing data, not zero performance.**
A line with `final otd% = 0`, `final dfr% = 0` and `otd variance` exactly `0` has no
delivery records flowing. It is a measurement gap. These lines are also **not flagged
To Review**, so they are invisible in the exception queue — and some of them carry real
spend. At Florida DC four such lines carry about $29.3k, the largest being Quat-Chem on
Nurofen Tablets 24 at roughly $24.5k.

Report them in a separate "blind spots" section with spend attached. **Never rank them as
underperformers** — telling a manager to chase a supplier who is actually at 100% but
unmeasured destroys trust in the whole report.

Distinguish them from genuinely dormant lines: zero variance **and** zero spend is simply
a supplier–product pair that isn't used. Not a finding.

**3. `To Review?` produces false positives.**
The flag fires on *category change* — including improvement — as well as on breach.
Expect flagged lines beating SLA on all three metrics (at Florida, Huhtamaki/Bottle at
+3.3 OTD and Quat-Chem/Gel Tube at +20). Call them out and recommend `Clear Review`. An
exception queue the manager stops trusting is worse than no queue.

Also check `Priority (Compound Alerts)`. In several model instances it is unpopulated on
every row, meaning the queue fires on category-change logic alone with **no
business-impact weighting** — which is precisely why a compliant $16k line and a failing
$171k line arrive looking equally urgent. Flag it as a configuration gap, because fixing
it changes every future queue.

**4. Negative Lead Time variance is good.**
Variance is actual minus contract, so negative means delivering *faster* than contracted.
Never list it as a breach. LT breaches are rare in this model — **if you find zero, say
so prominently.** "Every failure is OTD/DFR, not lead time" moves the supplier
conversation off planning horizons and onto reliability and capacity, which is a
different meeting.

**5. No Time, no Versions dimension.**
`KPI220` has neither. Values reflect the window configured in
**`KPI100 - Supplier KPI Time Settings`**. Never try to filter by period, never ask which
version the user wants. For a different horizon, point them at `KPI100`.

**6. Tier is a supplier-level attribute.**
`Final Supplier Tier` repeats across every row for a supplier — it does not vary by
product. So a Gold supplier can carry a line at 50% (trap 7 below), and a "Needs
Improvement" supplier can be near-perfect on everything except one SKU. Never infer
line-level health from tier.

---

## The analysis recipe

Run all of these on the single result set. They are free — the data is already in
context — and together they are what turns a data dump into a review.

1. **Spend at risk per supplier** — sum `Total Spend` across lines with negative OTD
   variance. Give both the absolute figure and the % of that supplier's total. **Rank by
   absolute spend at risk**, not by tier and not by worst percentage. A 50% line worth
   $700 is not a priority; a 71% line worth $65,000 is.

2. **Breadth vs depth** — breaching lines as a fraction of the supplier's active lines.
   Four of six failing moderately is a systemic constraint; one of six failing severely
   is an isolated SKU problem. Completely different conversations, so say which you see.

3. **The internal-inconsistency lever** — for each underperformer, find their 100% lines.
   This is the strongest negotiating position available: it removes "we can't" and leaves
   only "we aren't prioritising you". Always surface it when present.

4. **DFR-tracks-OTD test** — if fill rate degrades in lockstep with on-time rate, the
   constraint is upstream supply, not logistics; don't accept a freight-planning answer.
   If OTD falls but DFR holds, it's scheduling or transport. This tells the manager what
   to actually ask for.

5. **Cross-supplier product scan** — pivot breaches by product. If three or more
   independent suppliers miss the same SKU, the cause is almost certainly internal: spec
   change, demand volatility, late order placement, or volume spread so thin that nobody
   prioritises you. **Surface this above the individual supplier chases** — one internal
   fix can resolve more exposure than four phone calls. At Florida, four of five
   suppliers with data miss Nurofen Tablets 24, about $198k of breaching spend.

6. **Tier-masking check** — high-tier suppliers carrying material breaching spend. Tier
   is a weighted average, so a Gold supplier with two large 100% lines hides one at 71%.
   Flag these explicitly: they will **never** appear on a tier-based exception report, so
   without this check the manager never chases them. Shallow gaps on big spend are also
   the highest-probability quick wins.

7. **Reallocation arbitrage** — the same SKU where one supplier runs 100% and another
   runs poorly on comparable or greater spend, **at the same location**. Same site means
   no new qualification, so this is usually the highest-value single recommendation
   available and it falls straight out of the data.

---

## Output structure

Front-load the decision, keep evidence underneath. A procurement manager with ten minutes
needs the chase list, not the methodology.

```markdown
# [Location] — Supplier Performance Review

## The headline
[One number: spend at risk, $ and % of total. Then the single most important
structural insight — often what is NOT broken, because that reframes every
conversation they're about to have.]

## Chase these first — ranked by spend at risk
### 1. [Supplier] — $X at risk (N% of their spend)
[Breaching lines with OTD/DFR/spend. The leverage point. One-line diagnosis.]
[Top 3, ranked by absolute spend at risk]

## Two things you'd miss on a standard report
[Tier-masked suppliers. Cross-supplier product patterns.]

## Data gaps worth closing
[Blind-spot lines with spend. False-positive To Review flags. Unpopulated
alert weightings.]

## Your biggest single opportunity
[Reallocation arbitrage, with both sides' numbers.]
```

Close with two concrete follow-ups — typically a PO-level drill-down on the worst
supplier, and the same analysis at other locations to build a cross-site negotiating
position.

**Save the full detail as an artifact**: complete scorecard, every line with every
metric, per-supplier talk tracks, and an action plan split into this week / this month /
structural. Keep the conversational reply to decisions; put evidence in the artifact.
Add a second artifact — a spend-at-risk chart, breaching vs compliant vs unmeasured by
supplier — whenever there are four or more suppliers.

### Talk tracks

For each priority supplier write a quotable line the manager can take into a call. Not a
summary of the data — the actual argument. The pattern that works: **state their own best
performance, state their worst, name the gap, name the consequence.**

> "You're at 100% on our two smallest lines. On Nurofen Tablets 12 — $171,000 — you're at
> 62.5%, and on Dettol Antiseptic — $101,000 — you're at 50%. Your fill rate tracks your
> on-time rate almost exactly, so this is an allocation decision, not a transport
> problem. We need a written recovery plan in 30 days."

---

## Method note for the artifact

Close the artifact with short caveats so the numbers survive a supplier pushing back in
the room:

- Source module, grain, and the location slice used
- Variances read **as stored in the model**, not recomputed
- Spend-at-risk sums are the only arithmetic performed outside the model
- The ratio-roll-up limitation, and that `Total Spend` was validated against the roll-up
- No Time/Versions dimension; horizon is configured in `KPI100`

---

## Follow-ups in the same session

The cached coordinates are the whole point. Treat follow-ups as reads against data you
already hold:

- **"Tell me more about Henkel"** — answer from the result set in context. **No tool call.**
- **"What about our other DCs?"** — one `read_cells`, different `slice` value. Same line
  items, same `limit = 99`. Note that a multi-site read can exceed 99 rows, so either
  slice one site at a time or page with `skip`.
- **"Show me by product instead"** — re-pivot mentally. Same data. No tool call.
- **"What's the contracted target?"** — one read adding `Contract OTD%`, `Contract DFR%`,
  `Contract Lead Time`.
- **"Who changed this number?"** — `cell_change_history` with the slice copied verbatim.
  Calculated cells have no history of their own, so use `explain_cell` first to find the
  input that feeds it.

A five-question session should cost two or three tool calls, not fifteen.
