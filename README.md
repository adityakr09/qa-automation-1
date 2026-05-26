# Automation & QA Developer — Skills Assessment Submission

**Role:** Automation & QA Developer  
**Assessment date:** 25–26 May 2026  
**App under test:** Conduit (RealWorld demo) — `demo.realworld.io`

---

## Repository Structure

```
├── task1/
│   └── Task1_QA_Report.pdf          # Bug table (7 issues) + root-cause analysis
│
├── task2/
│   ├── Task2_Workflow.json           # n8n workflow — HackerNews Morning Brief
│   └── README_Task2.md              # API choices, transformation logic, error-path docs
│
├── bonus/
│   └── Bonus_UptimeMonitor.json     # n8n uptime monitor with 3-attempt retry + Discord alert
│
└── README.md                        # This file
```

---

## Task 1 — QA & Debug Report (Summary)

**App:** Conduit / RealWorld (`demo.realworld.io`) — React + Redux frontend, Node/Express backend

| # | Issue | Severity |
|---|-------|----------|
| 1 | Invalid email formats accepted on sign-up (no client-side validation, server error not surfaced) | **Critical** |
| 2 | Post-login redirect loses original destination — always lands on global feed | **High** |
| 3 | Stale/corrupt JWT silently fails — no 401 interception, no auto-logout | **High** |
| 4 | XSS via unsanitised Markdown rendering (`dangerouslySetInnerHTML` + no DOMPurify) | **Critical** |
| 5 | Pagination offset-based; no first/last controls; slow beyond page 10 | **Medium** |
| 6 | Deleted-article tags persist in sidebar indefinitely | **Low** |
| 7 | No account deletion — GDPR data-right gap | **Medium** |

**Root-cause deep-dive:** Bug #4 (XSS). Fix: wrap `marked(body)` with `DOMPurify.sanitize()` before passing to `dangerouslySetInnerHTML`. One-line change, eliminates a Critical security vulnerability.

---

## Task 2 — n8n Workflow (HackerNews Morning Brief)

**APIs:** HackerNews Firebase API (top stories → item detail — no auth key required)  
**Output:** Discord webhook embed  
**Logic:**
- Schedule trigger fires every hour
- Fetches top 500 story IDs → slices to top 5
- Enriches each ID with full story metadata (second API call)
- Aggregates + sorts by score
- IF branch: score ≥ 100 → 🔥 Hot Digest; < 100 → 📰 Quiet Digest
- All HTTP nodes use `onError: continueErrorOutput` → error formatter → Discord alert
- Credentials stored in n8n Credentials store (never hard-coded)

---

## Bonus — Uptime Monitor

**Target:** `demo.realworld.io` (the same app tested in Task 1)  
**Interval:** Every 5 minutes  
**Retry logic:** 3 ping attempts before alerting  
**Alert channel:** Same Discord webhook  
**Extra:** Response-time tracking per check; daily summary stub included (connect to Google Sheets for production use)

---

## Tools & Decisions

| Decision | Rationale |
|----------|-----------|
| HackerNews API (no auth) | Zero setup friction; always returns live data; free forever |
| Discord webhook output | Free, no extra accounts; familiar to dev teams |
| `onError: continueErrorOutput` on all HTTP nodes | Prevents silent crashes; routes every failure to a visible Discord alert |
| 3-attempt retry in uptime monitor | Avoids false positives from transient network blips |
| Score threshold = 100 | HN front-page baseline; easily tuned |

---

## How to Run

### Task 2 / Bonus in n8n
1. Install n8n: `npx n8n` or via Docker
2. Import the JSON files via **Workflow → Import from file**
3. Create a **Discord Webhook** credential in n8n Credentials store
4. Activate the workflow

### Task 1 (view report)
Open `task1/Task1_QA_Report.pdf` directly.
