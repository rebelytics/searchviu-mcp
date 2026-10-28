# Discovery — global reports, the data-sources registry and schema probes (§3)

Part of the **searchviu-mcp** skill (CC BY 4.0 — Eoghan Henn and contributors). Section numbers are global across `SKILL.md` and `references/` — the section map in `SKILL.md` says where each § lives.

**Load trigger:** Read before concluding that a report, a data class or a field does not exist — an empty `list_reports`, a question about the platform's inputs rather than its report output, a widget error from `get_report`, any first call through the REST passthrough, or any need to know which fields a data source can be filtered on. Also holds how to harvest global report ids.

## Contents

- 3. Discovery — establishing what exists before asserting it doesn't (routes 1–4; route 4 is the REST passthrough)

---

## 3. Discovery — establishing what exists before asserting it doesn't

A listing tool and a fetch tool are separate channels that can disagree
about what exists. An empty result only proves the query returned nothing.
Four routes, in the order they are usually needed.

**1. Global reports are invisible to `list_reports`.** Four empty listings
for one project (bare, plus three `search` terms) were followed by a
successful `get_report` on a report that project demonstrably has —
`isGlobal: true`, four widgets, a populated issue table. The evidence was
in the tool surface before the call: report objects carry an `isGlobal`
boolean, a field that only earns its place if `true` is reachable, and
`get_report`'s own parameter documentation describes `projectId` as
required "for global reports". So: **never convert an empty listing into
the claim that a project has no reports.** When the user names a report
you cannot enumerate, ask for its id or URL and fetch it directly — a
fetch-by-id exercises a different code path from the listing.

The operational bonus: because a global report is a template resolved
against whatever `projectId` you pass, **the same `reportId` runs against
any project by swapping `projectId` alone.** That is the standard way to
answer "does this data class exist at all, or just not here?" — running a
global report against a second, unrelated project distinguishes "absent
from the vendor's detector catalogue" from "absent from this project's
data", which no amount of querying inside one project can show.

**Harvest the ids from the application's own navigation.** The listing will
not enumerate global reports, but the UI has to: a project's `/reports` page
renders a sidebar link list covering both classes, so one pass over its
`<a href>` values — a single JS call in a browser tool, no clicking through
reports — surfaces every reportId the API refuses to list — each href has the shape
`/<projectId>/reports/<reportId>`, and the same pass covers the global URL
inventory and internal-linking reports as well as the AI Monitoring group.
Carry the general
form: **when a listing endpoint is known to be blind to a class of objects,
the application's own navigation is an enumeration channel.**

Harvested that way, the **AI Monitoring** group — the prompt-tracking output,
which the listing shows no trace of — comes back as a set of global reports:
Mentions, Share of Voice, Prominence, Recommendation, Citations, Top visible
brands, Prompt Tracking and New & Competitor Gap at last check. **Harvest the
ids afresh rather than keeping a list of them:** the report set is the
platform's to extend or rename, and a stored id list goes stale silently — an
id that no longer resolves reads as a missing report, and a report added
since reads as not existing. One sidebar pass per session costs one call and
is always current.

Being global, each of these runs against **any** project by swapping
`projectId` alone. Two behaviours are specific to them:

- **They ignore MCP filters.** A filter on `result_week_start_day` was
  accepted without error and changed nothing — every widget came back
  identical to the unfiltered call. On this report class
  `filtered == unfiltered` is the standing behaviour rather than a
  diagnosis, so do not try to scope them through the API at all: **pull in
  bulk and slice client-side.**
- **Bulk is the intended route, not a failure.** `Prompt Tracking` with
  `maxRowsPerWidget: 1000` returns every tracked prompt with its full answer
  text — on a mid-sized set ~6 MB, which overflows into a tool-results file.
  That overflow is the correct outcome here (see bulk extraction below), and
  it is what puts the answer text somewhere it can be parsed.

**2. The MCP reads report *output*; the platform's UI describes its
*inputs*.** This MCP wraps saved reports. It cannot see the platform's
data-sources registry — the per-project list of backing tables (order of
200) with field-level schemas, types, nullability, per-field descriptions
and a live row preview. Questions about a feature's input format, an
ingestion or configuration table, or "what fields does this data source
have" are addressed to that layer, and every MCP query about them
succeeds and returns nothing, which reads as absence rather than as
wrong-layer. **When a search across the entire report inventory comes
back empty, ask whether the question is even addressed to the layer being
searched before reporting a capability gap.** You cannot build a report
over a table whose fields you do not know, and an ingestion table has
nothing to report on yet — so schema discovery goes to the UI registry,
not to a new report. Two concrete uses: resolving field names and types
before building a report or an import file; and using the live row
preview as a positive control, which distinguishes "feature not yet
populated" from "query wrong". Caution: a row count on the registry's
overview tab is stale if the schema has never been refreshed — use the
live preview. And a nav entry's name can collide with a feature name
while meaning something else entirely; confirm what a section does before
treating it as the feature's home.

Two registry entries worth knowing before an image question is declared
unreachable, as observed at last verification. **`crawl_images` is the
image-level source for any image or alt-text question** — one row per image
occurrence with `url_complete`, `image_url`, `alt_text`,
`alt_attribute_present`, `width`, `height`, `format`, `file_size_bytes`,
`content_md5`, `phash`, `crawl_id` and `device`. It carries **no
`crawl_date`**: scope it by joining `crawl_id` to the URL source, never by a
date filter that the source will silently drop. And a name is not a
capability: **`image_classifications` is a sustainability-claims image
screening**, not a photo-versus-graphic classifier, so an "own graphics
inventory" or "which images are photos" question is not answered there. A
source remembered from another tool (a dashboard connector's "alt analysis"
table, say) may not be registered in the platform at all — confirm it in
the registry before building a report on it.

**3. An invalid filter field *sometimes* makes the validator enumerate the
schema — and sometimes is dropped in silence.** Read this route's caveat
first, then the route.

> **Caveat, at last check: the probe is no longer a reliable dimension
> test.** On some data sources an unknown dimension is now **silently
> ignored**: `get_report` returns every widget normally, with *unfiltered*
> totals, and no error text anywhere in the envelope. Observed with a
> deliberate nonsense dimension on a links report and, the following day, on
> a global content report in an unrelated project — while on a third report
> a valid filter applied correctly, so the behaviour is **per data source,
> not global**. Two consequences. (a) **A probe that returns unfiltered
> totals with no error has told you nothing about the schema** — not that
> the field exists, not that it does not. It has only told you that this
> source drops unknown dimensions. Never conclude "the field is valid
> because there was no error", and never conclude a report is healthy from
> the same silence. (b) Because a dimension name can no longer be validated
> by provoking an error, establish the field set another way: the platform
> UI's data-sources registry (route 2 above) carries the authoritative
> field-level schema per source, and a facet widget's values confirm a
> dimension is both present and populated. Where the error *does* come back,
> everything below still holds — it just cannot be relied on to come back.

Send `get_report` with a filter on an obviously invalid dimension name and
read the `Available fields` list out of the error:

```
Invalid filter fields: <bad_field>. Available fields: project_id, url_from,
url_to, ..., url_from_canonical_type, url_from_robots_tag, ...,
segment_pagetype, to_url_segment_pagetype, segment_pagetype_detail, ...
```

One such error surfaced ~30 filterable fields on a links report, several
materially useful and none visible in any widget's column config —
including a finer second segment group (`segment_pagetype_detail`) and a
canonical-type field on the link's source page. **A widget's displayed
column set is not its data source's field set**, so this is the only
reliable way to know what can be filtered before designing a query, and
it costs one call. The error is emitted once per widget, so pass
`maxRowsPerWidget: 1` when doing this deliberately. Pair it with the facet
rule below: the error gives you the field *names*, an unfiltered facet
list gives you a dimension's *values*.

**Read both of the error's lists — they blame different parties.** One
message can carry `Invalid fields: <name>` and `Invalid filter fields:
<name>` in the same string, and they are separate failure classes.
`Invalid filter fields:` names dimensions *your query* got wrong — that is
the probe above. `Invalid fields:` names fields the **report's own widget
config** still references after the data source dropped or renamed them; a
report in that state errors on every call, with or without filters, and the
fix is report-side, not query-side. Never debug your filter against an
`Invalid fields:` error. Two consequences: the deliberate invalid-filter
probe doubles as a free health check on the report's column config; and
because an extraction rename or replacement silently hard-breaks every
saved report whose widget still names the old field (observed when a field
was superseded by a `_schema`-suffixed successor), after any extraction
change re-run each dependent report once and read the error's first list.

Three limits of the enumeration, each learned the hard way:

- **It is a name list, not a typed schema.** Array-typed fields appear in
  it while accepting none of the scalar operators. Filtering
  `seo_issues` (an `ARRAY<STRING>`) with `like` passed validation and
  then failed in the engine with `Invalid cast from ARRAY<STRING> to
  STRING`. The two failure modes read differently and should be:
  an unrecognised name returns `Invalid filter fields: …` from the
  validator with the schema attached (the cheap discovery probe above);
  a recognised-but-mistyped field returns a BigQuery cast error from the
  engine — back out of that field entirely. The blast radius is
  **per-report, not per-widget**: one mistyped filter makes *every*
  widget return an error object, including widgets unrelated to the
  field, so an unexpected all-widgets-error response means suspect your
  filter before suspecting the report. Practical consequence: the array
  column is often the obvious route to "which URLs carry issue X" —
  plan to reach that data through a report whose widget already unnests
  it, never through a filter on the array.
- **It proves a field is addressable, not populated or current.** The
  listing is an accretion of every column the data source has ever
  carried, so a retired extraction stays in it indefinitely,
  indistinguishable from a broken one. A zero-row probe on an advertised
  field has three readings — populated but not matching, live but empty,
  or retired — and the API cannot distinguish them; only the person who
  configured the source can. Report an unpopulated advertised field as
  an observation with a question attached ("these return nothing —
  still in use?"), never as a defect finding. Where a working sibling
  field covers the same concept, retired is the leading hypothesis, not
  broken.
- **The names and labels travelling with it are claims, not behaviour**
  — see the report-metadata rule below.

**4. The REST passthrough reaches what no report exposes.** Beside the
report layer the server carries a passthrough to the platform's REST API
(§2). When `get_report` errors on one widget, or a data class seems
unreachable through any report, run `search_tools` with a few keywords for
the question before concluding anything — the top hits carry their input
schemas, so `invoke_api_tool` can follow directly. **A widget-level
`Widget does not have a data source configured` error is a custom widget
backed by its own endpoint, not an empty report**: the data exists and the
endpoint behind the widget answers. Observed on one project at last
verification: a global link-details report errored on its custom widget,
`search_tools` on "link details inlinks outlinks url" returned the endpoint
behind it, and the endpoint answered the question the report could not.
Known endpoints:

- `get_link_graph_url_details` ✅ — for one URL and a link direction:
  history across recent crawls, counts grouped by second-level directory,
  per-linking-page rows with `limit`/`offset`, added/removed rows against a
  compare crawl, and a click path. Read its rows with the field semantics
  in §10 before naming a link to a user. Paging past the first page ⬜ —
  not yet exercised; reconcile the paged row count against the reported
  total before trusting a full pull.
- `post_widgets_id_row_details` ⬜ — vendor-described row drill-down for a
  report widget; not yet exercised.

When an endpoint is used for the first time, report what it returned so its
entry can be promoted.

**A global report's field set bounds what it can answer.** The global
internal-linking report has no `url_to` field: a `url_to` filter on it is
dropped in silence (route 3 caveat), so it cannot name the pages that link
to a given URL — observed on one project at last verification. Ask that
question of `get_link_graph_url_details` (route 4) instead.
