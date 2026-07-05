# DebugProbe.AspNetCore — Proposed Additional Features

> Waterfall Timeline (Phase 1 & 2) is done. This document proposes the next set of features to build on top of it — what they are, why they matter, and how they'd fit into the existing architecture.

---

## 🎯 Why These Features?

Every feature below was picked using three filters:

1. **Reuses existing architecture** — no new infra, no external DB, same in-memory `DebugEntryStore` philosophy.
2. **High "aha!" moment** — the kind of feature a developer screenshots and shares with their team.
3. **Closes real gaps** — DebugProbe currently captures HTTP in/out beautifully, but has no DB visibility, no real-time push, no security story, and no way to share a trace with a teammate.

---

## 🗺️ Feature Overview

1. cURL Export + One-Click Replay
2. HAR File Export
3. Webhook Alerts (Slack/Teams/Discord)
4. Dashboard Security (Auth + IP Allowlist)
5. Exception Fingerprinting & Grouping
6. Trace Bookmarks, Notes & Shareable Export
7. Live Tail Dashboard (SignalR push)
8. EF Core / SQL Query Capture

---

## 1. cURL Export + One-Click Replay

**Why it matters:** Right now a developer can *see* an outgoing request in the waterfall, but if they want to test it themselves, they have to manually rebuild it in Postman. Generating a copy-pasteable `curl` command — and a "Replay" button that re-fires the exact request — turns DebugProbe from a *viewer* into a *tool*.

**How it works:** You already capture method, URL, headers, and body in `DebugOutgoingRequest`. This is pure string templating + one new endpoint that reuses `HttpClient` to resend the payload.

```mermaid
sequenceDiagram
    actor Dev as Developer
    participant UI as Dashboard (HtmlRenderer)
    participant API as /debug/replay/{id}
    participant Store as DebugEntryStore
    participant Ext as External Service

    Dev->>UI: Click "Copy as cURL"
    UI-->>Dev: curl -X POST ... (generated client-side from stored JSON)

    Dev->>UI: Click "Replay"
    UI->>API: POST /debug/replay/{entryId}
    API->>Store: Fetch original DebugOutgoingRequest
    API->>Ext: Re-send identical request
    Ext-->>API: New response
    API-->>UI: Show new response next to original (diff-style)
```

**Gotcha:** Replay is only truly safe for idempotent calls (GET). For POST/PUT (e.g. payment or order creation), replaying can re-trigger real side-effects — add a "⚠️ this will re-trigger the action" warning in the UI before firing.

---

## 2. HAR File Export

**Why it matters:** HAR (HTTP Archive) is the industry-standard format Chrome DevTools, Postman, and Charles Proxy all understand. Exporting a trace as `.har` means a developer can drag-and-drop it straight into tools their team already uses.

```mermaid
flowchart LR
    A["DebugEntry + DebugOutgoingRequest list"] --> B["HarExporter.cs"]
    B --> C{"Map to HAR schema"}
    C --> D["entry request"]
    C --> E["entry response"]
    C --> F["entry timings"]
    D & E & F --> G[".har JSON file"]
    G --> H["Download / Open in Chrome DevTools"]
```

**How it works:** Pure data-mapping from objects already in memory. No new capture logic needed.

---

## 3. Webhook Alerts (Slack / Teams / Discord)

**Why it matters:** Turns DebugProbe from "something I check" into "something that taps me on the shoulder." A simple rule engine — "notify if status ≥ 500" or "notify if outgoing call > 2000ms" — posts a formatted card to a webhook URL.

```mermaid
flowchart LR
    A["DebugEntry Saved"] --> B{"Matches Alert Rule?"}
    B -- "status is 500 or higher" --> C["Build Slack/Teams payload"]
    B -- "duration over threshold" --> C
    B -- "no match" --> D["Do nothing"]
    C --> E["POST to configured Webhook URL"]
```

**How it works:** One options class (`AlertRules`) + `HttpClient.PostAsync` on save. No new state to manage.

---

## 4. Dashboard Security (Basic Auth / API Key / IP Allowlist)

**Why it matters:** This is the feature a *maintainer* cares about even more than end users. Right now, anyone who can reach `/debug` can see full request/response bodies, including potentially sensitive data. Before this tool can be recommended for staging/prod use, it needs a simple, opt-in auth gate.

```mermaid
flowchart TD
    A["Request to /debug/*"] --> B{"RequireAuth enabled?"}
    B -- No --> F["Serve Dashboard"]
    B -- Yes --> C{"Valid API Key or Basic Auth header?"}
    C -- Yes --> D{"IP in Allowlist? (optional)"}
    D -- Yes --> F
    C -- No --> E["401 Unauthorized"]
    D -- No --> E
```

**How it works:** A small middleware check inserted before `DebugProbeMiddleware` continues to `HtmlRenderer` — same pattern ASP.NET Core apps already use everywhere.

---

## 5. Exception Fingerprinting & Grouping

**Why it matters:** When something breaks in a loop or under load, a developer currently sees 50 nearly-identical entries in the trace list. Grouping them by a "fingerprint" (hash of exception type + stack trace top frames) turns noise into a clean "This error happened 47 times" summary — similar to what tools like Sentry do, but built into your own app.

```mermaid
flowchart TD
    A["Unhandled Exception Caught in Middleware"] --> B["Compute Fingerprint: hash of ExceptionType + top 3 stack frames"]
    B --> C{"Fingerprint seen before?"}
    C -- Yes --> D["Increment counter on existing group"]
    C -- No --> E["Create new ExceptionGroup entry"]
    D & E --> F["Dashboard shows grouped list, e.g. NullReferenceException in OrderService x47"]
```

**Gotcha:** Deciding how many stack frames to hash needs some trial-and-error — too many frames makes groups overly specific, too few merges unrelated errors together.

---

## 6. Trace Bookmarks, Notes & Shareable Export

**Why it matters:** Ties directly into the existing **Trace Comparison View**. Right now traces disappear once the circular queue evicts them. Letting a developer "pin" an interesting trace, attach a note ("this is the race condition from ticket #482"), and export it as a portable `.json` bundle means it can be attached to a bug ticket or Slack message — and re-imported into someone else's DebugProbe instance for comparison.

```mermaid
sequenceDiagram
    actor Dev as Developer
    participant UI as Dashboard
    participant Store as DebugEntryStore

    Dev->>UI: Click "📌 Bookmark" + add note
    UI->>Store: Mark entry as Pinned (exempt from circular eviction)
    Dev->>UI: Click "Export"
    UI-->>Dev: Download trace-bundle.json
    Note over Dev: Attach to Jira ticket / send to teammate
    Dev->>UI: (teammate) Import trace-bundle.json
    UI->>Store: Load as read-only entry
    UI->>UI: Open in /compare against local trace
```

**How it works:** Mostly UI + serialization; the store already supports the shape of this data.

---

## 7. Live Tail Dashboard (SignalR Push)

**Why it matters:** Today the dashboard has to be manually refreshed to see new requests. A "Live Tail" mode (toggle switch, like `tail -f`) that streams new `DebugEntry` items the moment they're captured makes the tool feel alive — great for demoing or watching a flaky endpoint in real time.

```mermaid
sequenceDiagram
    participant MW as DebugProbeMiddleware
    participant Store as DebugEntryStore
    participant Hub as DebugProbeHub (SignalR)
    participant UI as Dashboard (Browser)

    MW->>Store: Save new DebugEntry
    Store->>Hub: OnEntryAdded(entry)
    Hub-->>UI: push "newEntry" event
    UI->>UI: Prepend row to table (no refresh)
```

**Gotcha:** `DebugEntryStore` is already a thread-safe circular queue — wiring a push-event system on top of it, and keeping multiple open browser tabs in sync, needs a bit more testing than a typical UI feature.

---

## 8. EF Core / SQL Query Capture

**Why it matters:** This is the **flagship feature** on this list. The outgoing-HTTP capture (via `DebugProbeHttpClientHandler`) already proves the "intercept and record" pattern works beautifully. Applying the same idea to database calls — via EF Core's interceptor API — lets developers see **SQL queries, duration, and even N+1 query detection** right next to the HTTP waterfall. This single feature elevates DebugProbe from "an HTTP debugger" to "a full request-lifecycle debugger."

```mermaid
sequenceDiagram
    participant API as Controller
    participant EF as EF Core DbContext
    participant Interceptor as DebugProbeDbCommandInterceptor
    participant Store as DebugEntryStore

    API->>EF: dbContext.Orders.ToListAsync()
    EF->>Interceptor: ReaderExecutingAsync(command)
    Note over Interceptor: Start Stopwatch, capture SQL + params
    Interceptor->>EF: Continue execution
    EF-->>Interceptor: ReaderExecutedAsync
    Note over Interceptor: Record duration, row count
    Interceptor->>Store: Attach DebugSqlQuery to current DebugEntry
    Store-->>API: (transparent — no behavior change)
```

**Bonus (optional, ship separately):** If the same normalized query template runs 3+ times inside one request, flag it visually as a likely N+1 issue.

**Gotcha:** Conceptually similar to the HttpClientHandler, but practically harder because:
1. EF Core's interceptor API differs slightly across versions (5/6/7/8).
2. Async streaming queries (`IAsyncEnumerable`) are trickier to time correctly than a simple request/response cycle.
3. The captured SQL data then needs to be visually merged into the *existing* waterfall timeline alongside HTTP calls — so it's also an integration task on top of the waterfall, not just a standalone capture feature.

---
