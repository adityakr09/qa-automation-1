# Task 2 – n8n API Integration Workflow

## Overview
**Workflow name:** HackerNews Morning Brief  
**File:** `Task2_Workflow.json`  
**Trigger:** Schedule — every 1 hour (configurable)

---

## APIs Used

| # | API | Why chosen |
|---|-----|------------|
| 1 | [HackerNews Firebase API](https://github.com/HackerNews/API) | Fully free, no auth key required, returns real-time ranked tech stories — ideal for a team "morning brief" |
| 2 | HackerNews item endpoint (same API, second call) | Enriches each story ID with full metadata: title, score, author, URL, comment count |
| 3 | Discord Webhook | Free, instant, no extra account needed; team-friendly notification channel |

---

## Workflow Architecture

```
[Schedule: every 1h]
       │
       ▼
[HTTP] GET /v0/topstories.json          ← First API call (500 IDs)
       │              │ (error)
       ▼              ▼
[Code] Slice top 5   [Error path] ──────────────────────────┐
       │                                                     │
       ▼                                                     │
[HTTP] GET /v0/item/{id}.json × 5       ← Second API call   │
       │              │ (error)                              │
       ▼              ▼                                      │
[Code] Aggregate &   [Error path] ──────────────────────────┤
       sort by score                                         │
       │                                                     │
       ▼                                                     │
[IF]  top score >= 100 ?                                     │
      │ TRUE          │ FALSE                                │
      ▼               ▼                                      │
[Code] Hot digest  [Code] Quiet digest                       │
      │               │                                      │
      ▼               ▼                                      │
[HTTP] Discord    [HTTP] Discord                             │
       │ (error)       │ (error)                             │
       └───────────────┴─────────────────────────────────────┤
                                                             ▼
                                                   [Code] Format error
                                                             │
                                                             ▼
                                                   [HTTP] Discord error alert
```

---

## Transformation Logic

The **"Transform – Slice Top 5 IDs"** Code node:
- Receives the raw 500-element integer array from the HN API
- Slices `ids.slice(0, 5)` — keeping only the 5 currently-top-ranked story IDs
- Emits one item per ID so n8n can loop through the next HTTP node in parallel

The **"Transform – Aggregate & Sort by Score"** Code node:
- Collects all 5 enriched story objects returned by the item endpoint
- Filters to `type === 'story'` (HN also has jobs, polls, comments)
- Sorts descending by `score` so the highest-scoring item is always first
- Bundles everything into a single output item for the IF node

---

## Conditional Branch (IF node)

**Condition:** `stories[0].score >= 100`

| Branch | Meaning | Discord embed |
|--------|---------|---------------|
| TRUE (≥ 100 pts) | Active news day | 🔥 Hot Digest — red accent colour |
| FALSE (< 100 pts) | Quiet day | 📰 Quiet Digest — blue accent colour |

The threshold of 100 was chosen because HackerNews front-page stories typically score 100–500+. A score below 100 in the top 5 usually indicates very recent or niche content.

---

## Output (Discord)

Each Discord message is sent as an **embed** with:
- Title indicating hot vs. quiet day
- Numbered list of up to 5 stories with score, comment count, author, and direct URL
- Footer timestamp showing when data was fetched

---

## Error Handling

| Failure point | Behaviour |
|---------------|-----------|
| HN top stories API fails | `onError: continueErrorOutput` → routed to error formatter → Discord alert posted |
| HN item detail API fails | Same — error output connected to formatter |
| Discord send fails | `onError: continueErrorOutput` → error logged in n8n execution history (avoids infinite loop) |
| Empty/malformed data | Code nodes throw explicit errors caught by n8n's error output chain |

No silent failures. Every error path produces a Discord notification with the failing node name, error message, and timestamp.

---

## Credentials Setup

1. In n8n, go to **Credentials → New → HTTP Header Auth**
2. Name it exactly: `Discord Webhook`
3. Set **Header Name** to `Content-Type` and **Header Value** to `application/json`
4. In each Discord HTTP node, set the URL field to your Discord webhook URL (Settings → Integrations → Webhooks in your Discord server)

> **Never hard-code the webhook URL in node parameters.** All nodes reference `$credentials.discordWebhookUrl` via the Credentials store.

---

## How to Import

1. Open your n8n instance
2. Click **+** (New Workflow) → **Import from file**
3. Select `Task2_Workflow.json`
4. Attach your `Discord Webhook` credential to all four HTTP Discord nodes
5. Click **Activate**
