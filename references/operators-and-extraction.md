# Operators, the missing ones, and bulk extraction (§11–§12)

Part of the **searchviu-mcp** skill (CC BY 4.0 — Eoghan Henn and contributors). Section numbers are global across `SKILL.md` and `references/` — the section map in `SKILL.md` says where each § lives.

**Load trigger:** Read before designing a filter that needs negation, OR, case sensitivity, prefix exclusion or paging past the 1,000-row cap, and before any bulk pull meant for programmatic processing or any export validated by its row count. Also holds the crawl-name filter patterns and the pull-to-file-then-jq workflow.

## Contents

- 11. Operators and the missing ones
- 12. Bulk extraction — overflow to file, sized above the threshold

---

## 11. Operators and the missing ones

Observed operator set: `eq`, `in`, `gte`, `lte`, `between`, `like`.
Multiple filters AND together. There is **no negation of any kind and no
OR**.

**Before designing any workaround for a missing operator, check whether
the platform already carries the answer** — in either of two hiding
places. As a **column on the row you are reading**: the datasets already
classify entities (page-type segments on both link endpoints, canonical
types, sitemap membership), so read the column rather than inferring the
classification from URL shape — URL patterns only separate page types
where the site separates them, and a plausible inference from a
familiar-looking pattern never announces itself as a guess. Or as a
**precomputed diagnostic elsewhere in the product**: issue catalogues,
quality flags and derived boolean columns are computed inside the
platform and are not bound by the API's operator set. The case-insensitive
`like` is structurally incapable of "find URLs containing a capital
letter", and the candidate workarounds (lexicographic range
complementation per path prefix; a 300k-row bulk export filtered locally)
were both unnecessary — the issues catalogue carried `uppercase-in-url`
with a per-crawl affected-URL count, answering the whole question in one
call, cross-crawl comparison included. A precomputed flag carries the
vendor's definition of the predicate and is computed per crawl, so it is
directly comparable across crawls; an ad-hoc filter is neither. The
workaround engineering is the *interesting* route and the precomputed
answer is the boring one, which is exactly why attention flows the wrong
way — ask "has someone already answered this?" before "can I express
this?".

- `like` is a **case-insensitive substring match with no word
  boundaries**. Short terms and terms that are substrings of common words
  will produce mid-word false positives that look like tool failure —
  under-specify the pattern, not the tool, is the first hypothesis.
- **Crawl names need a bounded pattern.** A `like` on a short crawl name
  also matches every longer name it prefixes (a week number matches the
  two-digit weeks that start with it). Bound the pattern with the delimiter
  that follows the name in the stored value — a trailing space on one
  project's naming convention — and read that convention off the unfiltered
  facet list first. A `like` on the date part of a name selects a whole
  month; compare named crawls with `in` over exact values. Confirm the
  selection on a small `maxRowsPerWidget` probe before the full pull.
- **Prefix exclusion via lexicographic range complementation.** On a
  sorted string column, the set of values starting with prefix P is the
  half-open interval [P, P′), where P′ is P with its last character
  incremented. The complement is two or three ordered ranges, each
  expressible with `lte` / `between` / `gte`, each AND-compatible. Pick
  boundary sentinels that cannot occur as real data.
- **Segment-based positive enumeration — the taxonomy IS the negation
  operator.** Where a data source exposes a page-type segment (on links
  data, for both endpoints: `segment_pagetype` for the source,
  `to_url_segment_pagetype` for the target), "all sources except class X"
  becomes `segment in [every class but X]`, which `in` handles natively.
  This is the cleanest substitute for the missing negation — and it works
  **only if X is its own segment.** In one case it was not: paginated
  listing URLs matched the parent listing's URL pattern, so pagination was
  folded into the listing segment, and "linked from a real category page"
  vs "linked only from page 7" — the two states the analysis existed to
  distinguish — were unseparable. The segmentation is therefore a hidden
  precondition of the query. **Check that the segmentation draws the
  boundary your question needs before designing a query around it**; a
  missing boundary is a cheap upstream fix to the segmentation config, not
  a limit of the query language. And
  **verify any URL pattern empirically with counting probes** rather than
  assuming the platform's documented convention — `?page=` can return zero
  where `?p=` returns hundreds of thousands of links, and an infinite-scroll
  variant can appear only under `&p=`.
- **Pagination via range subdivision.** With no offset parameter,
  subdivide the same ranges until every slice sits under the 1,000-row
  cap.
- **Verify the partition:** slice totals must sum exactly to an
  independently derived grand total — no gap, no overlap. An exact match
  also proves the partition complete.

## 12. Bulk extraction — overflow to file, sized above the threshold

For large pulls meant for programmatic processing, deliberately request a
result big enough to overflow the context limit so it lands in a
sandbox-readable file (the standard oversized-MCP-result technique). One
hard-won caveat: **a result just below the overflow threshold (~50–60 KB
payload) is not merely expensive — it lands in context, invisible to the
file-based pipeline**, and a harvest script reassembling slices from disk
will silently miss it. Size every slice comfortably above the threshold;
prefer fewer, larger slices and accept re-slicing a truncated one; verify
coverage by reconciling each slice's reported total against an
independently derived grand total, never by assuming the partition was
exhaustive.

**Pull once, then analyse the file locally.** Once a pull has landed in a
file, aggregate, group and compare it with `jq` or a script rather than
re-querying for each question — grouping by crawl, counting URLs per domain
or segment, summing a metric across crawls, a period-over-period table. Read
the envelope's shape first (§13 rule 2) and address the widget's rows by
that path; one example over a rows array:

```bash
jq '<rows-path> | group_by(.crawl_name) | map({crawl: .[0].crawl_name, urls: length})' pull.json
```

**Validate every bulk export on the distinct count of its key column, not
on its row count.** A CSV of tens of thousands of rows can match the
widget's reported `totalRows` exactly and read as complete while holding
several hundred fewer distinct URLs than rows, each duplicate appearing
twice. Every duplicate pair straddled a 1,000-row block
boundary and every duplicated group shared an identical sort-key tuple —
the signature of offset paging over a **non-unique sort ordering**: the
widget was sorted on two computed metrics with huge tie groups, the query
has no deterministic order within a tie, and each page re-orders the tied
rows independently. The dangerous half is invisible: because the total is
correct, every duplicate means **one distinct row silently dropped**,
and nothing in the file marks which. In a set difference against a
sitemap, those phantom absences inflate a genuine finding several-fold.

- If distinct count ≠ row count, the export is unusable — **do not
  de-duplicate it**, because the dropped rows cannot be recovered from the
  file.
- The remedy is upstream: **end the widget's sort on a unique column**
  (e.g. `url_to` ascending after the metric sorts), which makes the
  ordering total and fixes duplicates and omissions together.
- Sorting a widget on computed metrics with large tie groups is exactly
  the configuration that triggers this — and it is the same configuration
  the "ask for a sort instead of a new report" advice above produces, so
  request the tie-breaker in the same breath as the sort.
