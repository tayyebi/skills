---
name: dense-workspace-design
description: Use when converting a data-heavy application into compact, connected operational workspaces built for thousands of records. Covers UI density, task-based UX, bounded backend queries and APIs, search and pagination, concurrent editing, and bulk operations. Applies to admin consoles, inventories, queues, and other repeated record-management workflows.
---

# Dense Workspace Design

Build for the operator who must find, compare, change, and verify records all day with thousands of items in the system. Density means useful information and actions within reach, supported by data access and mutation paths that work at that scale.

This is a UI, UX, and backend redesign. A compact table backed by a full-collection download still has the original scaling problem.

## When to use

Use when an application has outgrown separate cards, long lists, oversized forms, or disconnected management pages; when moving between records requires repeated navigation; or when users need to work across large collections. Also applies when building a new operational workspace with those requirements.

Keep the existing product language, branding, framework, permissions, and business behavior unless changing them is part of the request. Grouping related entities on screen does not require merging their domain models. Scope changes to the requested workflows; a visual redesign does not authorize deployment or unrelated infrastructure changes.

## Start with the workload

Identify the actor, their frequent tasks, the records they compare, and what must remain visible while they act. Map the whole task: find a record, inspect related evidence, edit or act, verify the result, return to the same working context.

For each important collection, establish:

- Expected and peak size, growth, update frequency, and relationship fan-out.
- Common search/filter/sort combinations, selection scopes, and bulk actions.
- Current list/detail requests, payload sizes, query counts, rendering costs, and write paths.
- Business invariants such as processing order, destination precedence, access boundaries, and retained-history limits.

Use realistic thousands-scale fixtures when no target is supplied; 10,000 primary records is a useful starting scenario, with related records and skewed distributions. It is a test scenario, not a universal capacity promise. Define practical latency and resource budgets for the application's environment.

Implement the interface and the data paths together. If access is limited to a prototype or frontend, make that limitation explicit and define the missing backend contract rather than claiming the result handles production scale.

## Organize around connected workspaces

Group navigation by operator tasks. Put related records and their operational context together:

| Task | Main area | Adjacent context |
|------|-----------|------------------|
| Configure routing | Searchable rules or routes | Destinations, conditions, impact, simulation |
| Investigate an incident | Active incidents | Rule, entity evidence, related deliveries |
| Recover failed work | Filtered queue | Selected job, attempts, destination, retry controls |
| Manage inventory | Searchable inventory | Selected item, relationships, recent activity |

Use these as examples, not prescribed domain names. A useful default shell is a compact collapsible sidebar, a page heading with primary actions, a search/filter toolbar, and a dense record table. Use a second pane for selection details, editing, or related records when side-by-side work reduces navigation. Keep routine tasks in the workspace and make expensive diagrams, diagnostics, and analysis explicit destinations or actions.

Represent filters, sorting, page/cursor, selected record, and view mode in navigable URL state where practical. Direct links, reload, browser Back, and returning from an editor should restore the working context. Keep sensitive search terms out of shareable URLs when the domain requires it; never encode secrets or unsaved form bodies there. Save URLs through browser history deliberately so typing a query does not create dozens of Back steps.

## Make density readable

Prefer aligned rows when users compare records across the same fields. Use cards for genuinely different content or narrow-screen summaries, rather than one large card per entity by default.

- Give each row a strong identity, a few decision-relevant columns, and a clear status. Move long bodies, raw JSON, large histories, and rare fields into details.
- Use consistent small spacing and restrained typography. As adaptable desktop starting points, try 32–40 px rows, 13–14 px primary text, and a 4/8/12/16 px spacing scale. Preserve readable type, zoom, and usable control targets; fit more useful structure before shrinking text.
- Align numeric values, use stable column widths where comparison matters, and expose full values when truncating. Essential information must be reachable by keyboard and touch, not only hover.
- Use status text alongside color, visible focus, labeled controls, and semantic tables. Add grid-style keyboard behavior only when implementing its complete interaction model.
- Use available width, sticky headers where helpful, and contained horizontal scrolling for wide tables. On small screens, prioritize columns or use compact summaries, stack panes, and keep actions reachable. Avoid page-wide sideways scrolling.
- Provide distinct loading, empty collection, no matches, partial failure, unavailable, and permission states, each with a useful next action.

## Make the collection contract scale

Pagination must happen before records reach the browser. Every large list and relationship picker needs a bounded query path; a paged main table with a dropdown containing 10,000 options is unfinished.

### Reads

- Apply authorization and tenant scope before search, counts, pagination, and relationship lookup. Filter and sort the full authorized collection on the backend, then return a bounded page.
- Return row summaries. Fetch full records, bodies, histories, and relationship pages on demand. Do not fetch the full configuration or all entities to render the shell, calculate browser-side totals, or open an editor.
- Enforce a maximum page size, allowlisted sort/filter fields, and bounded input. A default of 50 with 25/50/100 choices often works; adapt it to row cost and the product.
- Sort deterministically using a stable unique tie-breaker. Prefer cursor/keyset pagination for large changing feeds; offset paging can suit bounded collections needing page numbers. Changing the query resets incompatible pagination state. Cursors must belong to the same query and authorization scope.
- Match storage access to observed workloads: indexes for hot filter/sort combinations, joins or batched enrichment for visible rows, and precomputed summaries only when justified. Avoid per-row requests, repeated full scans for each row's relationships, and synchronous global analysis on every page load.
- Make count semantics explicit: total matches, approximate matches, current-page rows, and retained-history totals are different facts. If exact counts are expensive, use a cheaper labeled count or next-page availability rather than blocking the list.

A generic contract, adapted to the existing API, might be:

```text
GET /records?q=...&status=...&sort=updated_desc&limit=50&cursor=...
  -> {items: [row summaries], next_cursor, count: {value, kind, scope}}
GET /records/{stable_id}
  -> {record, revision, related_counts}
GET /records/{stable_id}/related?limit=50&cursor=...
  -> {items: [related summaries], next_cursor}
PATCH /records/{stable_id}  {changes, expected_revision}
  -> {record, revision}
```

Do not introduce a new database or search service solely to imitate this shape. An existing configuration store can expose bounded summaries and targeted writes. Measure its actual lookup, scan, and persistence costs at the target size and report remaining limits honestly.

### Browser behavior

Debounce search, cancel superseded requests when supported, and ignore responses for obsolete queries. Keep loading and errors local to the affected pane. Bound rendered rows; add virtualization only when measured rendering costs justify its focus, accessibility, and scroll-restoration complexity. Virtualization does not replace backend pagination.

Refresh only relevant data. Preserve selection, scroll, keyboard focus, and drafts during updates. A changing feed must not reorder a selected row under a click; pause automatic repositioning or offer an explicit new-results action. Invalidate affected summaries after a successful write without reloading every collection.

## Separate inspection, editing, and operations

Selecting a row or relationship inspects it. Editing changes a draft. A save or named operational action changes the system. Make the current mode and commit point clear.

Use explicit Save/Cancel forms as the default for consequential configuration. If established autosave is appropriate, make pending, saved, failed, and conflict states visible and provide recovery. Preserve entered values on validation or network failure. Use dedicated editor routes or an adjacent editor pane for long or contextual tasks; reserve dialogs for short interactions that benefit from interruption.

Address records with immutable management identities, never display indexes, sort positions, or labels. Distinct records with the same label must remain independently editable. A legacy dataset lacking safe identifiers may need a separate management key; preserve runtime identities and reference semantics during that migration.

Persist targeted changes with an expected revision, ETag, or equivalent concurrency check. A stale editor receives a conflict and a refresh/reconcile path that retains the draft. Apply related changes transactionally or atomically; a failed persistence attempt must not leave live memory claiming an unsaved value. Return the committed result and record the actor, action, and affected scope through the existing audit mechanism.

Route web, API, and other management clients through the same validation and persistence rules. Hiding an action in the UI cannot enforce its eligibility or authorization.

## Make bulk scope explicit

Keep these scopes distinct in both UI and API:

| Scope | Meaning |
|-------|---------|
| This page | The selected visible records only |
| Selected records | Explicit stable IDs, possibly across pages |
| All matching | The server-defined result of the active filters |

Display the scope and count before a consequential operation. When filters change, clear selection or visibly retain it with its original scope; never silently reinterpret old selection as a new query. Do not load every matching record into the browser to implement Select all.

For disruptive or destructive actions, provide a compact review of the operation, scope, and important exclusions. Keep routine reversible edits lightweight. Bind an all-matching review to a defined target set: for example, a server-side snapshot token or an as-of boundary with explicit eligibility rules. If targets change materially, require a refreshed review; a live query must not silently sweep in newly matching records after confirmation.

On execution, recheck authorization and action eligibility. Use a transaction when all-or-nothing behavior is required; otherwise report succeeded, skipped, failed, and retryable items. Use set-based updates, bounded batches, or a background job according to operation cost. Large jobs need progress, retry semantics, duplicate-submit protection, and a link to results that survives navigation. Offer cancellation only when the backend can honor it, with honest partial-completion behavior.

## Validate the complete task at scale

Use isolated representative fixtures and existing test tools. Include both small collections and the target thousands-scale dataset, with repeated labels and sort values, long text, missing optional values, skewed statuses, and high relationship fan-out. Verify meaningful behavior:

- Search finds a record absent from the loaded page. Relationship pickers find items beyond their initial results.
- Sorting and paging preserve identity and stated consistency semantics. Rapid query changes never display a stale response as the latest result.
- An operator can inspect, edit, save, and return to the same list context using a direct link and browser Back.
- A concurrent change produces a recoverable conflict; a failed save retains the draft and leaves persisted state unchanged.
- Bulk selection works across pages. All-matching actions include eligible off-page records, exclude unauthorized records, and honor the reviewed scope when the dataset changes.
- Live refresh does not erase forms, move focus, lose selection, or interrupt the task.
- Keyboard use, zoom, and narrow layouts preserve access to essential information and actions.
- Response size and rendered row count stay bounded as collection size grows. Measure first and deep-page latency, related-query count, and bulk execution cost; investigate N+1 growth and hidden full-collection work.

A screenshot demonstrates layout. A scale claim also needs observed data access and workflow behavior. If the environment prevents a check, state what was inspected, what remains unverified, and the bottleneck or dependency involved.

## Pitfalls

- Shrinking fonts and margins while preserving disconnected pages and repetitive navigation.
- Fetching everything, then filtering, paging, or counting in JavaScript.
- Bounding the main table but leaving pickers, relationship panels, exports, or overview counts unbounded.
- Saving an entire stale configuration document to change one visible record.
- Rendering every relationship edge or running overlap analysis before the user requests it.
- Polling the whole workspace so an operator loses a draft or clicks a moving target.
- Treating a UI reorganization as permission to change routing, ordering, retention, or authorization semantics.
- Reporting success because a ten-record demo looks compact.

## Checklist

- [ ] Frequent operator tasks and business invariants are identified; related context stays reachable throughout each task.
- [ ] Tables and panes are compact, readable, accessible, and navigable through URLs and browser history.
- [ ] Lists and pickers use bounded server queries, stable ordering, and summaries; details load on demand.
- [ ] Counts, selection scopes, and retained-data limits communicate what they actually describe.
- [ ] Targeted writes use stable identities, concurrency checks, atomic persistence, and recoverable errors.
- [ ] Bulk execution honors reviewed scope, current authorization, eligibility, and partial-result semantics.
- [ ] Representative thousands-scale behavior and resource costs are verified, with remaining limits stated.

Related references, when needed: [user journeys](../user-journey-engineering/SKILL.md) for deeper task mapping, [search and pagination](../search-filtering-pagination/SKILL.md) for storage-specific query techniques, and [responsive design](../responsive-mobile-first-design/SKILL.md) for adapting interactions to smaller screens.
