# searchviu-mcp

Open-source **companion skill for the searchVIU MCP server** — `list_projects`, `list_reports` and `get_report` over the searchVIU crawl data warehouse. Load it into any MCP-capable AI agent (Claude, Cursor, Codex, n8n, etc.) alongside a connected searchVIU MCP.

## What this skill does

Field-tested guidance for querying the report-wrapper API: the mental model (a thin wrapper over saved report widgets, bounded by the report inventory), how to discover what exists before asserting it doesn't, the crawl-scope rules, `totalRows` as the counting primitive and its units, why aggregate widgets recompute inside the filter, facet self-exclusion, field semantics, the operator set and the range arithmetic that substitutes for the missing ones, bulk extraction, troubleshooting, and a pre-flight checklist to run before delivering any figure.

## When it triggers

Querying searchVIU data through an MCP — sizing or counting report rows, filtering by crawl date, building prefix exclusions, paging past the row cap, deciding whether a data class is reachable at all, `totalRows`, `get_report`, the AI Monitoring global reports, or a searchVIU report that shows different numbers in the UI and through the API.

## What this repo contains

- `SKILL.md` — the mental model, the tool surface, the epistemics for querying an opaque API, the pre-flight checklist, and a section map saying when to load each reference file.
- `references/discovery.md` — global reports and how to harvest their ids from the application's navigation, the data-sources registry, and the invalid-filter schema probe (§3).
- `references/scope-and-counting.md` — crawl scope, counting and aggregate widgets (§4–§6).
- `references/reading-reports.md` — derived columns, report metadata, facet widgets and field semantics (§7–§10).
- `references/operators-and-extraction.md` — the operator set, the missing operators and bulk extraction (§11–§12).
- `references/troubleshooting.md` — UI/API disagreements, session loss, the per-project release gate and the performance-data level behind it (§14–§15).
- `LICENSE` — CC BY 4.0.

## Install

Install the whole skill directory — `SKILL.md` and `references/` must travel together — into your MCP-capable agent's skills directory, following the client-specific path, and restart the client. Tool names in the skill are written without the MCP server prefix; match by the suffix.

## Related

[`searchviu-tracking-strategy-builder`](https://github.com/rebelytics/searchviu-tracking-strategy-builder) builds and reports on a prompt tracking set for searchVIU's AI visibility feature and uses this skill for the read side.

## Contributing

Open an issue or a PR at [github.com/rebelytics/searchviu-mcp](https://github.com/rebelytics/searchviu-mcp).

## Credits

- Original author: [Eoghan Henn](https://www.rebelytics.com) / [LinkedIn](https://www.linkedin.com/in/eoghanhenn)
- Affiliation: Eoghan Henn is a co-founder of searchVIU but does not actively work for the company. This skill is maintained independently and is not an official searchVIU product.

## License

CC BY 4.0 — see [LICENSE](./LICENSE). Use it, fork it, adapt it, monetise it. Keep the attribution.
