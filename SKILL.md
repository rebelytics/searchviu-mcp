---
name: searchviu-mcp
description: "Companion skill for the searchVIU MCP server (list_projects / list_reports / get_report over the searchVIU crawl data warehouse). Load whenever querying searchVIU data through an MCP — sizing or counting report rows, filtering by crawl date, building prefix exclusions, paging past the row cap, or deciding whether a data class is reachable at all. Also load on mentions of totalRows, get_report, searchVIU reports via API, the AI Monitoring global reports, or when a searchVIU report shows different numbers in the UI and through the API. Teaches the real behaviour of the report-wrapper API — the unfiltered-aggregate trap, count units, facet self-exclusion, and the range arithmetic that substitutes for missing operators."
version: 1.2.1
license: CC-BY-4.0
metadata:
  author: Eoghan Henn and contributors
  source: https://github.com/rebelytics/searchviu-mcp
---

# searchVIU MCP — Companion Skill

Field-tested guidance for driving searchVIU's crawl data warehouse through
its MCP server. Every behavioural claim in this skill was observed against
real reports; nothing is inferred from vendor documentation alone.

**Coverage boundary.** This skill covers **querying the report-wrapper API** —
`list_projects`, `list_reports`, `get_report`, their filters and operators,
counting, scoping, paging and bulk extraction. It does **not** cover building
or configuring reports and widgets in the searchVIU UI, crawl setup and
scheduling, page-type segmentation config, or searchVIU as a product. The
boundary runs through prompt tracking rather than around it: its **output** is
in scope, reachable through the global AI Monitoring reports catalogued in
Discovery route 1 (§3), while only its **inputs** — the `prompts` and
`competitors` tables — are UI-only and belong to
`searchviu-tracking-strategy-builder`. Silence
in this file is not evidence a capability is absent: a companion skill is
loaded precisely because the agent doesn't know the tool, so an uncovered
surface reads as a missing feature when it may only be an uncovered surface.
Before concluding something cannot be done, re-read mental-model rule 1 (§1) — the
usual answer is a report that doesn't exist yet, not an API limit.

---

## 1. Mental model

The MCP exposes exactly three tools: `list_projects`, `list_reports`,
`get_report`. There is no query language behind them — `get_report` is a
**thin wrapper over saved report widgets**. Two consequences drive
everything else in this skill:

1. **Capability is bounded by the report inventory, not the API — and the
   inventory is user-editable.** If a data class appears unreachable, the
   far likelier explanation is that no saved report exposes it. Creating a
   report over that data source in the searchVIU UI takes minutes, and the
   MCP reads it immediately. Never report "not reachable through the API"
   without stating whether the report set is fixed or extensible — the
   honest finding is "nothing currently exposes this; here is what to
   add", and the first remedy is a new report, not a vendor escalation.
   **Two preconditions before invoking this rule** (see Discovery below):
   the listing may be blind to the report class you are looking for, and a
   question about the platform's *inputs* rather than its report output is
   not addressed to this layer at all. Establish both before recommending
   that the user build something.
2. **The widget's UI defaults are part of the UI, not the API.** The API's
   true default is "no filter", which silently changes the population
   being measured (see the crawl-scope rule below). Before trusting any
   figure pulled through the MCP, reproduce one known number from the UI
   through the API.

`get_report` caps at 1,000 rows per widget, offers no GROUP BY, COUNT,
aggregation, or offset parameter. All of those gaps have workarounds
below.

## 2. Tool surface

Marker key: ✅ = exercised in real sessions, behaviour observed. ⬜ = not
yet exercised — treat the row as vendor-described, verify before relying
on it. When you use a ⬜ item, report back what it actually did so the
entry can be promoted.

| Tool | Status | Notes |
|------|--------|-------|
| `list_projects` | ✅ | Enumerates projects; IDs feed `list_reports`. At last check each project also carries `mcpEnabled` (boolean) and `mcpAccessHint` — the per-project **release gate** (see Troubleshooting). Read `mcpEnabled` before calling the other two tools on a project; it is the only place the gate is visible short of an error. |
| `list_reports` | ✅ | Enumerates **project-owned** reports only. Every report it has ever returned carried `isGlobal: false`, and a global report confirmed to exist in a project never appeared in that project's listing. Never read an empty listing as "this project has no reports". Errors with `This project is not released for MCP access` on a project whose `mcpEnabled` is `false`. |
| `get_report` | ✅ | Returns all widgets of one report. 1,000-row cap per widget via `maxRowsPerWidget`. Accepts filters (see operator set). Every filtered response carries `totalRows` per widget. For a **global** report, `projectId` must be passed alongside `reportId` — it resolves the template's data source to that project. Subject to the same per-project release gate as `list_reports`, and to the performance-data level below. |
| Performance-data level (per project, inside the release) | ⬜ | A released project carries a second gate on the project's MCP settings page, **"Performance data"**, with three values: **Off** (performance values withheld), **Levels** (the default: coarse buckets such as none/low/medium/high or top 3/top 10; revenue and conversions only as present/absent; no filtering on these columns; no exports) and **Exact values**. It governs clicks, impressions, positions, sessions and revenue; crawl data is unaffected. The in-app AI assistant has its own identical switch, set separately. Known from the settings page text only — not yet observed in a `get_report` payload, and `list_projects` has not been seen to expose it. The next session that reads a performance column through the MCP records what a Levels-gated value looks like in the row, what a filter on that column returns, and whether `list_projects` reports the level, then promotes this row. |

## Reference files — load on demand

This skill uses progressive disclosure. This file holds what applies on every
invocation — the mental model, the tool surface, the epistemics for an opaque
API and the pre-flight checklist. Everything else lives in `references/` and is
read when its episode comes up. The load triggers are mandatory, not
suggestions: a crawl-scoped figure pulled without the scope rules, or an
absence reported without the discovery routes, is exactly the confidently
wrong output this skill exists to prevent. If you notice an episode was
handled without its reference loaded, treat that as a failure to report, not
a shortcut that worked.

- `references/discovery.md` (§3) — **read before concluding that a report, a data class or a field does not exist — an empty `list_reports`, a question about the platform's inputs rather than its report output, or any need to know which fields a data source can be filtered on. Also holds the report ids of the global AI Monitoring reports.**
- `references/scope-and-counting.md` (§4–§6) — **read before the first crawl-scoped `get_report` call of a task, and before quoting any count, total or aggregate-widget value — the crawl-date rules, `totalRows` as the COUNT primitive and its units, call serialisation, multi-section reports, and why aggregates recompute inside the filter.**
- `references/reading-reports.md` (§7–§10) — **read before interpreting a share, ratio or rating column, a report's name or description, a facet list, a sentinel value, an inlink count, or an extraction field used as a join or grouping key — before reading dates off an AI Monitoring row — and before a crawler-recorded status code, console error or rendering difference is reported as a site defect.**
- `references/operators-and-extraction.md` (§11–§12) — **read before designing a filter that needs negation, OR, case sensitivity, prefix exclusion or paging past the 1,000-row cap, and before any bulk pull meant for programmatic processing or any export validated by its row count.**
- `references/troubleshooting.md` (§14–§15) — **read when the UI and the API disagree on a number, on any `Server not initialized` or `This project is not released for MCP access` error, and when a performance column (clicks, sessions, revenue) returns labels instead of numbers or cannot be filtered — before retrying anything.**

## Section map — where each § lives

Section numbers are global across the file set. A cross-reference written as
"above" or "below", or naming a section by its heading ("the crawl-scope
rule", "Discovery route 1", "the bulk-extraction rule"), resolves through
this table.

| § | Heading | File |
|---|---|---|
| 1 | Mental model | `SKILL.md` |
| 2 | Tool surface | `SKILL.md` |
| 3 | Discovery — establishing what exists before asserting it doesn't | `references/discovery.md` |
| 4 | Crawl scope — the single most important filter | `references/scope-and-counting.md` |
| 5 | Counting — totalRows is the aggregation primitive, with units | `references/scope-and-counting.md` |
| 6 | Aggregate widgets recompute inside the filter | `references/scope-and-counting.md` |
| 7 | Figures that don't reconcile — suppress rather than caveat | `references/reading-reports.md` |
| 8 | Report metadata is a claim, not a spec | `references/reading-reports.md` |
| 9 | Facet widgets are self-excluding — never read absence off a facet list | `references/reading-reports.md` |
| 10 | Field semantics — sentinels, the inlink_count family, and extraction keys | `references/reading-reports.md` |
| 11 | Operators and the missing ones | `references/operators-and-extraction.md` |
| 12 | Bulk extraction — overflow to file, sized above the threshold | `references/operators-and-extraction.md` |
| 13 | Epistemics for querying an opaque API | `SKILL.md` |
| 14 | UI and API disagree — reconcile, and suspect the newest config first | `references/troubleshooting.md` |
| 15 | Troubleshooting — `Server not initialized` is session loss, not a blip | `references/troubleshooting.md` |
| 16 | Pre-flight verification — run before delivering any figure | `SKILL.md` |

## 13. Epistemics for querying an opaque API

These six rules are not specific to searchVIU — they hold for any remote
data source you cannot see inside, and they are the ones most often
skipped. Each is stated here in the form it bites in this tool.

1. **Run an unfiltered control call before reading anything into an
   absence.** A filtered empty result and a data source with no content
   are indistinguishable from the response alone. Here the failure mode is
   sharpened by the crawl-scope and empty-multifilter signatures above:
   zero rows in every widget means the scope is absent, zero rows in one
   means the filter selected nothing.
2. **Inspect the actual response shape on first call; never assume the
   envelope before scripting against it.** Widget payloads differ by
   widget type, `rows` can be empty on a widget whose config promises
   values (the `multifilter_bar` case above), and metric cards and data
   tables in one envelope can disagree.
3. **Treat `filtered == unfiltered` as evidence the filter was ignored**,
   not as evidence the segment equals the whole. In this MCP that reading
   is doubly necessary because facet lists legitimately show unfiltered
   values for the dimension being filtered — so check the data table's
   `totalRows`, not the facet.
4. **Keep full-payload pulls small and prefer count views where only a
   number is needed.** `totalRows` with `maxRowsPerWidget: 1` is the count
   view here — but see the multi-section note: on a wide report even a
   one-row probe carries every widget's config, so budget probes by widget
   count, not by row count.
5. **A scope you did not set is still a scope, and it appears in neither
   the request nor the response.** The crawl-scope rule above is the
   headline case — omitting `crawl_date` silently aggregates across every
   stored crawl — but the general form is what to carry: a default cannot
   be found by re-reading the call (it isn't there) or by inspecting the
   result (it isn't there either); the only detector is varying the
   parameter deliberately. Before quoting any figure, issue the same call
   twice with the scoping parameter at its extremes and compare; if the
   number moves, state the resolved scope beside it. `totalRows` is a
   report on the filter set you **sent**, never on what the warehouse
   holds, and an exhausted paging loop is the most dangerous case because
   having reached the end supplies the feeling of completeness that would
   otherwise prompt the check. A per-row date that predates your filter —
   a `first_seen`-style column, an older `crawl_date` in a cross-crawl
   set — is no evidence at all about how wide the scope was.
   And a scope you *did* set is still an aggregate that hides recency —
   a distinct failure from the unset default above, because here the scope
   was chosen, correct and stated in the deliverable. Any rate or share read
   off a single `crawl_date` — the 4xx proportion of a pagetype, the share of
   URLs carrying a canonical or indexability problem, a redirect-chain ratio
   — answers "how often, in that crawl" and is read as "how often, now", and
   the two diverge exactly when the thing has been fixed since, which is the
   case where acting on the number is most wasteful. The cross-crawl form is
   worse: an unfiltered aggregate smears an issue that existed in two old
   crawls across the entire stored series, so a resolved problem keeps
   reporting a non-zero rate indefinitely. **Band it before it carries a
   recommendation.** Re-issue the identical call once per `crawl_date` across
   the recent crawl series — `maxRowsPerWidget: 1` and read `totalRows`, one
   cheap probe per crawl — and read the sequence rather than the aggregate;
   on any surface that takes a since-date instead of a snapshot handle, the
   equivalent is nested windows differenced against each other, one call per
   band. Two things belong in the deliverable: which crawl the finding
   actually lives in, and whether it is still present in the most recent
   *populated* crawl — positive-control that crawl first (crawl-scope rules
   above), because an unprocessed newest crawl returns zero and reads as
   "fixed". The sequence also recovers the mechanism the aggregate destroyed:
   a count non-zero in the older crawls and exactly zero in every crawl since
   some date names both the cause and roughly when it stopped. Two riders:
   check the denominator, because a dramatic percentage over a handful of
   URLs is not a finding; and where a widget attaches a headline rate to a
   named pagetype, segment or example URL, read it as scoped to that row
   until proven otherwise, not as a site total that happens to be displayed
   beside an example.
6. **Test an identifier before using it as a join key across two calls.**
   A field shaped like a key is an implicit promise about persistence, and
   the promise is untested until the same scope is fetched twice and the
   values compared — this MCP already supplies two counterexamples: the
   UI's crawl UUIDs are not the warehouse's `crawl_id` values (crawl-scope
   rule 2), and an extraction field can hold a localised display label
   rather than the stable ID beside it. Key any before/after comparison on
   a **natural key** — `url`, `url_to`, `url_from` — and diff the
   substantive fields explicitly rather than inferring change from set
   membership. An unstable key manufactures a difference in exactly the
   operation used to detect difference, so the instrument's noise arrives
   looking like the signal, and it misleads in both directions: a real
   change would have been believed for the wrong reason.

## 16. Pre-flight verification — run before delivering any figure

Before reporting numbers derived through this MCP, check:

1. **Crawl scope:** does every crawl-scoped query carry `crawl_date`, and
   was that filter shown to *bite* on this data source — an impossible
   date returning zero, or facet totals matching the known crawl size?
   Where the source has no `crawl_date` column, is the figure labelled as
   scope-inferred rather than filtered? Is any unfiltered number labelled
   as a cross-crawl aggregate? For any per-URL list, is the resolved crawl
   date stated as its as-of date, and was a sample live-checked if the
   crawl is not fresh?
2. **Unit:** for every count, is the source widget named and the unit
   (distinct entities vs rows) identified?
3. **Control:** was at least one known UI figure reproduced through the
   API in this session?
4. **Null results:** for any "none found", did a positive control confirm
   the entity class exists in the report's data source at all — run as
   the identical call against a known-populated scope, not merely
   unfiltered? For a recently finished crawl, was its completion time
   checked before reading zeros? For "no report exposes this", was the
   listing capable of seeing that report class, and is the question
   addressed to the report layer at all?
5. **Partitions:** do slice totals reconcile exactly against an
   independent grand total?
6. **Attribution:** was every `get_report` call issued serially, and does
   at least one returned row agree with every filter that was sent?
7. **Aggregates:** for any figure from an aggregate widget, are the active
   filters stated, and does none of them constrain the dimension being
   aggregated over?
8. **Sections:** were metric cards from different sections of one report
   summed only after a probe proved the sections disjoint?
9. **Derived columns:** does every share, ratio or rating move in the
   direction the arithmetic requires when the query scope is varied? If
   not, suppress it.
10. **Exports:** does the distinct count of the key column equal the row
    count, and does the paging sort terminate on a unique column?
11. **Metadata claims:** was any scope or definition taken from a report
    name, description, or column label confirmed against the returned
    rows rather than quoted as-is?
12. **Extraction keys:** for any figure grouped or joined on an
    extraction field across markets, was the field probed in a
    non-default locale to confirm it is a stable key and not a localised
    label?
13. **Defaults:** was every scoping parameter varied to confirm the figure
    is not an artefact of a scope nobody chose, and is the resolved scope
    stated beside the number?
14. **Diff keys:** was any before/after comparison keyed on a natural key
    (URL) rather than a row id or row order, with the substantive fields
    compared explicitly?
15. **Domain boundary:** for any inlink, orphan or linking-dependence
    figure from a multi-domain project, were same-domain and cross-domain
    sources separated, and is the figure labelled with which one it is?
16. **Performance-data level:** for any clicks, impressions, positions,
    sessions, revenue or conversions figure, was the project's MCP
    "Performance data" level confirmed to be **Exact values**? Under the
    default **Levels** the value is a bucket label, not a number, and the
    column cannot be filtered — a filter on it that returns nothing, or a
    column of words where numbers were expected, is this gate, not an
    empty scope. The level is set by a project admin per project (and
    separately for the in-app AI assistant); "released" says you can read
    the project, not at what resolution. Where the level is not Exact,
    label the figure as a level, never convert it to a number, and hand
    the user the settings path rather than a workaround.
17. **Crawler-recorded defects:** for any status code, console error,
    failed request or rendered-vs-raw difference about to be reported as a
    site defect or ranked as a lever, was it reproduced once with the client
    the fix is for — a plain HTTP client for a status code, a clean browser
    starting from empty storage for a rendering value — and is the finding
    labelled with what reproduced and what did not (§10)?

A delivery that fails a check gets corrected, not caveated.

## Feedback on this skill

If the user finds the methodology wrong or incomplete, or hits searchVIU MCP
behaviour this skill doesn't cover, offer to draft an issue for
https://github.com/rebelytics/searchviu-mcp, checking the existing issues and pull
requests for the same problem first; if the problem is the agent not
following the skill's rules, acknowledge and correct it instead.

searchVIU MCP — Companion Skill: Eoghan Henn and contributors | CC BY 4.0
