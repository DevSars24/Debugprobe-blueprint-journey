# PR Guidance: Pin/Favorite Trace + Redaction Preview Toggle

**Target repo:** `DebugProbe/DebugProbe.AspNetCore`
**Suggested branch:** `feature/pin-and-redaction-preview`
**Suggested PR title:** `feat: pin traces + redaction preview toggle`

This PR bundles Phase 2 features **#9 (Pin/Favorite a Trace)** and **#10 (Redaction Preview Toggle)** into one PR, following the repo's established pattern of shipping 2 related-but-independent small features together (see PR #162, #149, #151).

---

## 1. Why this pairing works

Both features pass the maintainer's three filters from the Phase 2 proposal:

1. **Zero new infra** — no EF Core, no external DB, everything stays in the existing in-memory `DebugEntryStore`.
2. **Immediate developer value** — usable the moment it ships, no config needed for #9; opt-in config for #10.
3. **Reuses existing data** — both build directly on `DebugEntry`, no new capture pipeline.

Keep them as **separate commits** in the same PR so either can be reviewed/reverted independently if the maintainer pushes back on one.

---

## 2. Feature #9 — Pin/Favorite a Trace

### Goal
Prevent an important trace from being evicted from the bounded `DebugEntryStore` queue during an active debugging session, and surface it clearly on the dashboard.

### Files to touch

| File | Change |
|---|---|
| `DebugEntry.cs` | Add `public bool IsPinned { get; set; }` |
| `DebugProbeExtensions.cs` | Register new minimal endpoint `POST /debug/pin/{id}` (toggle) alongside existing routes |
| `DebugEntryStore.cs` | Update eviction logic to skip pinned entries when trimming to `MaxEntries`, **or** cap max allowed pinned entries |
| `HtmlRenderer.cs` | Render a separate "Pinned" section at the top of the index page, regardless of queue position |
| `debugprobe-ui.js` (if applicable) | Wire pin/unpin button click → calls the toggle endpoint |

### Implementation checklist
- [ ] `IsPinned` defaults to `false`
- [ ] Toggle endpoint flips the flag and returns updated state (no auth changes needed beyond existing `AuthorizationPolicy` gating)
- [ ] Eviction logic explicitly excludes pinned entries from trim — write a unit test that fills the store past `MaxEntries` with a pinned entry present and asserts it survives
- [ ] Decide and document a cap on pinned entries (e.g. hard limit or a config option) to avoid unbounded memory growth if a user pins everything
- [ ] Pinned section renders above the normal entry list, visually distinct (icon/badge)
- [ ] **No persistence** — confirm state resets on app restart, consistent with the rest of the in-memory model (don't accidentally wire this to any storage)
- [ ] Unit tests: `DebugEntryStoreTests` (eviction skip), `HtmlRendererTests` (pinned section renders), endpoint test (toggle round-trip)

### Things the maintainer will likely check
- Does eviction still respect `MaxEntries` overall, or can pinned entries cause unbounded growth? Have an explicit answer/test for this.
- Is the toggle endpoint idempotent-safe and covered by the existing `AuthorizationPolicy` when configured?

---

## 3. Feature #10 — Redaction Preview Toggle

### Goal
Let a developer temporarily reveal the original (pre-redaction) value of a header/query param/JSON field on the trace details page — **only in local/development environments**, and only when explicitly opted in.

### Files to touch

| File | Change |
|---|---|
| `DebugProbeOptions.cs` | Add `public bool AllowRedactionPreview { get; set; } = false;` |
| `DebugEntry.cs` / capture pipeline | Store both redacted and original values in memory when `AllowRedactionPreview` is enabled (only then — don't retain originals otherwise) |
| `HtmlRenderer.cs` | Add toggle UI on the details page; **only rendered** (not just disabled) when `EnvironmentUtils` confirms local/dev **and** `AllowRedactionPreview` is true |
| `RedactionUtils.cs` | **No changes** — redaction pipeline itself stays untouched; this is purely an additional rendering path |

### Implementation checklist
- [ ] `AllowRedactionPreview` defaults to `false` — explicit opt-in required
- [ ] Toggle button is **completely absent** from HTML output outside dev environments (not `disabled`, not hidden via CSS — actually not rendered), so its existence isn't signaled in production output/source
- [ ] Original values are only retained in memory when the flag is on — verify no behavior change (no extra memory retention) when the flag is off
- [ ] Confirm this cannot be enabled via any code path when `AllowUiInProduction` is also relevant — double-check the two flags don't interact in a way that leaks preview into production
- [ ] Unit tests: option defaults to false; toggle absent when flag off; toggle present + functional when flag on + environment is dev; redacted values still stored correctly in `RedactionUtils` path (unchanged)
- [ ] Document the trade-off explicitly in README: enabling this means original sensitive values are held in memory for the trace's lifetime

### Things the maintainer will likely scrutinize hardest
This repo's redaction work (`#106`, `#111`) was done directly by the maintainer, not a contributor — treat this as the most security-sensitive part of the PR:
- Be very explicit in the PR description about **exactly** where original values are stored, for how long, and under what conditions.
- Call out that `RedactionUtils` is untouched — this is an additive rendering path only, not a redaction bypass.
- Consider adding a code comment at the storage point flagging it as security-sensitive, so future changes don't accidentally widen exposure.

---

## 4. PR description template

```markdown
## Summary
Implements Phase 2 features #9 (Pin/Favorite a Trace) and #10 (Redaction Preview Toggle).

## #9 Pin/Favorite a Trace
- Adds `IsPinned` flag on `DebugEntry`
- New `POST /debug/pin/{id}` toggle endpoint
- Pinned entries are excluded from `DebugEntryStore` eviction and shown in a
  dedicated "Pinned" section at the top of the dashboard
- In-memory only, no persistence beyond process lifetime

## #10 Redaction Preview Toggle
- New `AllowRedactionPreview` option, defaults to `false`
- When enabled AND environment is local/dev, adds a toggle on the trace
  details page to temporarily reveal original (pre-redaction) values
- `RedactionUtils` is unchanged — this is purely an additive rendering path
- Toggle UI is not rendered at all (not just disabled) outside dev environments
  or when the flag is off

## Testing
- [ ] Unit tests added for eviction skip logic
- [ ] Unit tests added for pin toggle endpoint
- [ ] Unit tests added for redaction preview gating (env + flag combinations)
- [ ] Manually verified toggle is absent from rendered HTML in production mode

## Scope note
Both features stay within existing `DebugEntryStore` / `HtmlRenderer` / `DebugEntry`
architecture — no new external dependencies, no database, no infra added.
```

---

## 5. Bonus / additional ideas (optional, same PR or follow-up)

All pass the same three filters (zero infra, immediate value, reuse existing data). Good candidates if you want to add a third small feature or queue up the next PR:

| Idea | What it needs | Notes |
|---|---|---|
| **Export trace as JSON** | Button on details page, reuse existing entry serialization | Same pattern as existing cURL export (`#146`) |
| **Notes field on a trace** | Free-text `string? Note` on `DebugEntry`, in-memory only | Mirrors Pin implementation almost exactly — could even go in *this* PR as a third commit if scope allows |
| **Duplicate/Retry count badge** | Group existing entries by normalized route + method, count | No new capture, pure aggregation over `DebugEntryStore` |
| **Slowest N requests filter** | Sort/filter existing entries by latency (already captured) | UI-only, no new data needed |

**Recommendation:** If you want a third feature in this same PR, add **Notes field** — it reuses the exact same `DebugEntry` + in-memory + no-persistence pattern as Pin, so it won't raise new review concerns and keeps the PR coherent ("small in-memory annotations on a trace" as a theme) rather than scope-creeping into analytics/export territory.

---

## 6. Pre-submit checklist

- [ ] Separate commits per feature (pin / redaction preview / optional notes)
- [ ] All new options have safe defaults (`IsPinned = false`, `AllowRedactionPreview = false`)
- [ ] No changes to `RedactionUtils.cs` redaction logic itself
- [ ] No new external dependencies in `.csproj`
- [ ] Tests added for eviction, toggle endpoint, and env/flag gating
- [ ] README updated with new options and their trade-offs
- [ ] PR description follows the template above, explicit about security trade-off of #10
