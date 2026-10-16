# UI/API disagreements and troubleshooting (§14–§15)

Part of the **searchviu-mcp** skill (CC BY 4.0 — Eoghan Henn and contributors). Section numbers are global across `SKILL.md` and `references/` — the section map in `SKILL.md` says where each § lives.

**Load trigger:** Read when the UI and the API disagree on a number, on any `Server not initialized` or `This project is not released for MCP access` error, and when a performance column (clicks, sessions, revenue) returns labels instead of numbers or cannot be filtered — before retrying anything.

## Contents

- 14. UI and API disagree — reconcile, and suspect the newest config first
- 15. Troubleshooting — `Server not initialized` is session loss, not a blip

---

## 14. UI and API disagree — reconcile, and suspect the newest config first

The UI and the MCP are two independent readers of the same store; either
can fail alone, in either direction. Observed both ways: a correct UI
number next to a misleading unfiltered API aggregate, and a just-built
report rendering blank in the UI while the API returned it in full — the
cause was a config error in the report that only the UI was strict about.
When the two disagree, the newest thing in the chain (usually the report
configuration just written) is the leading suspect, not the vendor's
rendering layer — and a tolerant consumer succeeding does not vouch for
the artefact's correctness.

## 15. Troubleshooting — `Server not initialized` is session loss, not a blip

HTTP 400 `Bad Request: Server not initialized` (JSON-RPC error code
`-32000`), arriving on every call after the connector has already answered
normally earlier in the same task, means the **server-side MCP session is
gone** — and it cannot be re-initialised from inside the task. Two things
that look like remedies are not: a tool-list refresh reports success while
changing nothing, because it only re-reads the tool list over the existing
connection rather than re-dialling; and the user reconnecting the connector
does not reach the already-running session. The fault is **per-session, not
server-side** — validated by the same server answering normally in a
parallel new task while the old session went on failing.

So **stop after one retry** rather than spending a user round-trip on
attempts that cannot work. Save whatever build or parsing scripts the pull
was to feed, plus a short handoff note naming the exact calls still
outstanding, to the workspace, and ask the user to resume in a new task. On
resumption, confirm the connector with a one-row probe
(`maxRowsPerWidget: 1`) before committing to the full pull. That pattern is
validated end to end: the probe answered on the first call, the full bulk
pull then succeeded and fed the saved scripts unchanged. The rule holds for
any MCP server, not only this one.

**`This project is not released for MCP access` is a per-project admin
setting, not session loss and not a credentials fault.** As of the platform
version observed when this rule was written, every project sits behind a
release gate: `list_reports` and `get_report` return the error above, with
the release URL appended (`A project admin can release it at
<platform>/<projectId>/settings/mcp`), until a project admin has released
that project — and `list_projects` reports the gate per project as
`mcpEnabled` with the same URL in `mcpAccessHint`. Three things distinguish
it from the session-loss case above: it arrives on the first call of a
task, not after the connector has already answered; `list_projects` keeps
working; and the error names a remedy that is neither a retry nor a new
task. Two things it does *not* mean: that the connector is misconfigured,
and that the project was never reachable. **Access that worked before can
be withdrawn by a platform change** — at last check, projects queried
without incident the previous day all came back `mcpEnabled: false` the
next, with no change on the account's side — so a project's reachability
is a fact about today, not a property of the connector. The order of
operations: read `mcpEnabled` off `list_projects` first; on `false`, stop,
hand the user the release URL from `mcpAccessHint` (the gate is an admin
action inside the platform UI, which nothing in this MCP can perform), and
resume once they confirm. Do not spend retries, tool-list refreshes or a
new task on it — none of those touch a setting on the project.

**A release carries a second, finer gate that produces no error at all.**
The same settings page holds a "Performance data" level — Off, Levels
(the default) or Exact values — governing clicks, impressions, positions,
sessions, revenue and conversions, with crawl data unaffected (tool-surface
table, §2). Under Levels a performance column comes back as a bucket label
and cannot be filtered, so the symptoms are a column of words where numbers
were expected, or a filter on that column that selects nothing — both of
which read as an empty scope or an ignored filter unless the level is
checked first. When a performance figure looks wrong on a released project,
ask the project admin which level is set before treating it as a data or
filter problem; the fix is a settings change on their side, not a different
query. The exact row and error shapes under Levels are not yet observed —
record them when first seen.
