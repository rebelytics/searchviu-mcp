# Crawl scope, counting and aggregate widgets (§4–§6)

Part of the **searchviu-mcp** skill (CC BY 4.0 — Eoghan Henn and contributors). Section numbers are global across `SKILL.md` and `references/` — the section map in `SKILL.md` says where each § lives.

**Load trigger:** Read before the first crawl-scoped `get_report` call of a task, and before quoting any count, total or aggregate-widget value — the crawl-date rules, `totalRows` as the COUNT primitive and its units, call serialisation, multi-section reports, and why aggregates recompute inside the filter.

## Contents

- 4. Crawl scope — the single most important filter
- 5. Counting — totalRows is the aggregation primitive, with units
- 6. Aggregate widgets recompute inside the filter

---

## 4. Crawl scope — the single most important filter

- **Every crawl-scoped query must carry an explicit `crawl_date` filter**
  (`crawl_date eq YYYY-MM-DD`). An unfiltered `get_report` aggregates
  across **every stored crawl** — observed as a roughly six-fold
  overstatement of a 4xx count that read as a plausible single-crawl number with no
  marker distinguishing it. The UI's crawl selector defaults to the latest
  crawl; that default does not exist through the MCP.
- **A `crawl_date` filter that changes nothing was ignored, not satisfied
  — and the call still succeeds.** The rule above has **no enforcement**
  on a data source that lacks the field: at last check an unknown
  dimension is silently dropped on some sources (Discovery route 3
  caveat), so `get_report` returns a plausible, *unscoped* number with no
  error. Observed: a `crawl_date gte <date>` filter on a links data source
  returned totals byte-identical to the unfiltered call (eight-figure row
  count, unchanged to the digit), because that source has no `crawl_date`
  column at all. **Prove the filter can bite before believing anything it
  produces:** issue it once with a value that *must* return zero — a date
  later than the most recent crawl — and confirm you get zero. A filter
  that returns the same total for a valid date and an impossible one is
  not filtering. A second, cheaper positive control where the report has
  facets: check that the multifilter facet totals sum to the crawl size
  you already know from the crawl-management UI. One whole report class is
  known to behave this way for *every* filter: the AI Monitoring global
  reports (Discovery route 1) accept a filter and return the unfiltered
  result, so there the only correct move is to stop filtering and slice
  client-side.
- **Where the source genuinely lacks `crawl_date`, the scope is inferred,
  and must be labelled as such.** Link-level sources commonly carry no
  crawl-date column while their URL-level siblings do. The workable route
  is indirect: resolve the crawl date on a sibling source that *does*
  carry the field, confirm exactly one crawl falls in the window, and rely
  on the report template's own crawl pin (a report named for the last
  crawl is pinned to it). Then say so in the deliverable — the links
  figure's scope comes from the report's pin and a sibling source's date,
  **not** from a filter you applied. Never present it as filtered.
- **Unfiltered calls on large URL-level sources hit a 30-second engine
  timeout.** Observed: `BigQuery execution failed: Job execution was
  cancelled: Job timed out after 30 sec` on an unfiltered call that
  answered instantly once a `crawl_date` filter was added. So on those
  sources the crawl-date filter is not only the correctness requirement —
  it is what makes the report answer at all, and a bare timeout is a
  prompt to add scope rather than evidence the report is broken. Note the
  asymmetry this creates: on a source **with** the field, omitting it
  fails loudly; on a source **without** it, supplying it fails silently.
- **`crawl_date` is the only reliable crawl handle.** The crawl UUIDs
  shown in the UI's crawl-history list are not the warehouse's `crawl_id`
  values — filtering on them returns nothing.
- **Distinguish "no data for this crawl" from "no matching rows".** If a
  crawl-date filter returns zero rows in *every* widget — no URLs of any
  pagetype, not just no errors — the crawl scope itself is absent from
  that report's data source. That is the empty-multifilter signature. A
  genuinely clean crawl still shows its population.
- **A crawl offered in the UI's crawl selector can hold zero rows in the
  report's data source.** The selector is populated from the crawl
  registry, the widgets from the processed data store — two systems with
  a lag between them — so a scope can be selectable, labelled "Latest",
  and empty. Observed: a crawl that finished the same morning rendered
  `0` in every widget ("total redirects 0", "No data available"), which
  reads as a spotless site; the previous crawl's scope returned dozens of
  issue rows on the identical call. The empty-scope and clean-scope cases have
  byte-identical presentations, and the failure always renders as good
  news. The required check is a **positive control on a known-populated
  scope using the identical call** — not merely an unfiltered call,
  which can succeed on older data and disguise the gap. Timing
  heuristic: a crawl that finished within hours of the query is the
  leading suspect for an unprocessed scope, so check its completion
  time before interpreting its numbers.
- **Resolve the crawl's date before shipping anything per-URL.** The same
  crawl supports conclusions with different shelf lives: structural
  findings — the composition of pagetypes, the shape of the link graph —
  survive weeks of drift, while a list of specific URLs is a claim about
  their state *now* and decays from the moment the crawl finished. A report
  named "Last Crawl" is not a date: read the actual `crawl_date` /
  `crawl_name` value out of the response and state it in the deliverable as
  the list's as-of date. Before delivering a per-URL remediation list off a
  crawl that is not fresh, live-check a stratified sample and report the
  observed stale rate, or re-pull against a newer crawl scope. An
  unqualified URL list implicitly claims to describe the present, which the
  data may never have supported.
- Report templates may pin their crawl selector to one crawl scope (e.g.
  weekly full crawls). If another scope (daily/live crawls) is needed,
  that is a missing-report situation, not an API limit — create a report
  over that data source (mental-model rule 1).

## 5. Counting — totalRows is the aggregation primitive, with units

The API has no COUNT, but **every filtered `get_report` response carries a
`totalRows` field for the filtered set**. Issuing a filter with
`maxRowsPerWidget: 1` and reading `totalRows` is a one-row-cost COUNT.
Set arithmetic over cheap counts answers questions the row cap appears to
forbid — a handful of probes can reduce a link graph of millions of rows to
the few thousand that matter, without paging a single full page.

**A count is only meaningful with its unit attached:**

- A **data table's** `totalRows` is a **distinct-entity count** — the
  table de-duplicates by its key column (typically URL).
- An **aggregation widget's** counts are **row counts** (observations).
- The two agree only when the filtered set has no repeats — i.e. exactly
  the single-crawl query someone would use to sanity-check the method.
  Observed divergence on multi-crawl sets: order of magnitude.

Rules: state which widget a number came from; read both and reconcile — a
ratio near 1 means the filter selected one crawl, well above 1 means the
set spans crawls; never validate a counting method on a query where the
candidate units coincide.

**Serialise every `get_report` call — never two in one turn.** Probes are
cheap, which makes firing several at once tempting; do not. Two parallel
`get_report` calls against the same report crossed: one returned
`tool "get_report" timed out after 180s`, and the other returned the
*timed-out* call's result — sample row matching the other filter, metric
cards carrying the other filter's counts while the data table in the same
envelope reported `totalRows: 0`. Re-running the filter alone returned a
plainly correct result, so the fault was concurrency, not the filter. The
response does not echo the request back, so **the returned rows are the
only available proof of which query was answered.** Recognise a crossed
response by either tell: a sample row whose values contradict a filter you
sent, or metric-card counts that disagree with the data table's
`totalRows` in the same envelope. The timeout is the visible half of the
failure; the silently mis-attributed, entirely plausible number is the
expensive half.

**One filtered call on a multi-section report is a cross-section
measurement, not a single count.** A report with several data tables, each
carrying its own baked-in section filter, re-evaluates *every* widget
under any filter you add — so one probe simultaneously answers "how many
rows in section A, B, C, D, E match this condition". Use that
deliberately: filtering an "internal links to non-indexable pages" report
on one destination host revealed a six-figure number of links appearing in
**both** the noindex table and the non-self-canonical table, i.e. two headline metrics
measuring the same links, with nothing in the report's presentation
signalling the overlap. **Never sum metric cards from different sections
of one report without a probe proving the sections are disjoint.** Note
the token cost: `maxRowsPerWidget: 1` does *not* make a probe cheap on a
wide report, because the response carries every widget's configuration
regardless (~6k tokens on a 22-widget report). Two well-chosen probes beat
six narrow ones; when the question is compositional rather than a single
count, prefer the deliberate-overflow bulk pull below.

## 6. Aggregate widgets recompute inside the filter

An aggregate/grouped widget's values are computed **within the applied
filter, not per entity**. A widget keyed on `url_to` with columns
`total_inlinks` / `inlinks_from_paginated` / `inlinks_from_non_paginated`
returns, say, `14000 / 700 / 13300` for one product unfiltered; adding a
source-side filter (`url_from like "?p="`) changes the same product's row
to `600 / 600 / 0` (illustrative figures, real shape). That is arithmetically correct within the filtered
set and catastrophic if read as a property of the product — the widget
existed to find targets whose non-paginated count is zero, and any
source-side filter manufactures that condition for every row. The output
looks like the answer and is an artefact of the query, and because it is
internally consistent nothing in the response marks the change.

- **Never apply a filter on one side of a relationship while reading an
  aggregate over the other side.** For any figure taken from an aggregate
  widget, state which filters were active when it was computed and confirm
  none of them constrain the dimension being aggregated over.
- **Filters accept only base data-source fields, never a widget's computed
  columns.** Filtering on `inlinks_from_non_paginated` — a column the
  widget itself returns — fails with `Invalid filter fields`. The widget
  can *compute* the condition you care about but cannot *select* on it, so
  the selection has to move upstream into the report's saved config: a
  baked-in filter, or a sort order that puts the wanted rows in the first
  page. **Ask for the sort first** — it is the cheaper request and avoids
  needing a new report. (See the export-validation rule below before
  paging over such a sort.)
