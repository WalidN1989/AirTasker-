# Airtasker MCP — Lead Discovery Proof of Concept

> **Status:** ✅ MCP connection verified, pulling live data. Discovery works.
> **Date:** 2026-07-15 · **Owner:** Walid (Digital Urgency, AU)
> **This repo:** notes + baseline data from the POC. **No app has been built yet — by design.**

---

## 1. What this is

A proof-of-concept to answer one question:

> *"If someone posts a task on Airtasker that matches our expertise, can Claude Code discover it through the **official** Airtasker MCP and flag it as a relevant opportunity?"*

**Answer: Yes — for discovery.** See constraints in §4 before planning anything.

Business context: [digitalurgency.com.au](http://digitalurgency.com.au/) — we are a **service provider (Tasker)** that wins digital work (WordPress, SEO, automation, CRM, integrations) by **bidding on customer-posted tasks**. Leads CRM: `digitalurgency-crm-production.up.railway.app/app/leads`.

---

## 2. The official MCP (what we connected to)

- **Endpoint:** `https://mcp.airtasker.com/mcp` (remote, HTTP transport, **OAuth 2.0**)
- **Official docs:** https://support.airtasker.com/hc/en-au/articles/58505062391321-Setting-up-the-Airtasker-MCP-Server
- **Rule for this project:** official MCP only — **no scraping, no browser automation, no unofficial APIs, no invented tools.**

### The 7 tools it exposes (confirmed live)

| Tool | Type | What it does |
|---|---|---|
| `browse_tasks` | read | Search the **public** task feed (keyword, geo, online/physical, sort, paginate) |
| `get_task` | read | Full details of one task (description, budget, dates, poster slug) |
| `get_profile` | read | Public profile of a poster/tasker (rating, tasks run, etc.) |
| `get_my_tasks` | read | Tasks **you** posted (customer side) |
| `get_offers` | read | Offers **on tasks you posted** (customer side) |
| `post_task` | write | Post a task as a buyer |
| `accept_offer` | write | Hire + pay a tasker for **your** task |

In the desktop connector, every tool is set to **"Needs approval"** — nothing runs without an explicit click. Keep it that way.

---

## 3. How to connect (setup notes)

### On a full Claude Code CLI (your home Mac) — easiest
```bash
# Option A — official plugin (recommended by Airtasker)
/plugin marketplace add airtasker/airtasker-mcp-server
/plugin install airtasker@airtasker
/mcp        # completes OAuth in the browser

# Option B — manual
claude mcp add --transport http airtasker https://mcp.airtasker.com/mcp
/mcp        # completes OAuth in the browser
```

### On the Claude **desktop app** (what we used in the office) — fallback
The desktop app has **no `/plugin` and no `claude` CLI**. Instead:
`Settings → Connectors → Add custom connector → URL: https://mcp.airtasker.com/mcp → Connect → OAuth login`.
Tools then appear automatically to the Claude Code agent.

> OAuth is a **human step** — the AI cannot log in for you. Never print/commit tokens.

---

## 4. ⚠️ The critical finding (this changes the model)

**The MCP is built for the CUSTOMER side, not the PROVIDER side.**

Our model is provider/bidding, but there is:
- ❌ **No `make_offer` / `submit_quote`** — you **cannot** place a bid via the MCP.
- ❌ **No messaging tool** — you **cannot** chat with the client via the MCP.
- `get_offers` / `accept_offer` are the **opposite direction** (you *posting* a job and *hiring*), not you winning work.

**Implication:** the MCP is a **discovery + qualification engine only.** The actual **bid and client chat stay MANUAL in the Airtasker app.** Trying to replicate Airtasker's bidding/messaging is out of scope and unsupported.

Also note: `browse_tasks` returns the **public, non-personalised feed** — same for everyone. The OAuth login links the *account tools* (`get_my_tasks`, etc.), but discovery is not personalised.

---

## 5. The realistic architecture — a **lead radar**, not an auto-bidder

Our edge is **speed**: fresh tasks have **0–5 offers**; stale ones have 50–130. Being an early bidder wins. So the MCP's job is to catch matches **minutes after posting**.

```
Airtasker public feed
      │   MCP browse_tasks  (sort=recent, low offersCount, keyword-filtered)
      ▼
Scoring filter  (freshness × competition × budget × category × buyer rating)
      ▼
Digital Urgency CRM  ── lead lands, ranked, with a drafted offer to copy
      │   push notification → phone
      ▼
Tap lead → deep-link opens the task in the Airtasker APP
      ▼
You bid + chat with the client   (manual, human)
```

**Design principle: discovery/qualification is automated; the bid + conversation are human. Never auto-submit offers.**

---

## 6. What we observed (secondary notes)

- **Freshness:** near real-time — newest task in the feed was posted minutes earlier.
- **Sorting:** `recent`, `price_asc`, `price_desc`. **Recency = default and key signal.**
- **Location:** `lat`/`lon`/`radius_km` supported; **no category filter** (use `search_term` + read `categoryName` back).
- **Online vs physical:** `task_types` filter + `isOnline` flag.
- **Budget:** `priceDollars` + `isFixedPrice` (often a placeholder — confirm with `get_task`).
- **Competition read:** `offersCount` per task (huge for prioritisation).
- **Reference:** `slug` per task = its ID (feed into `get_task`; likely maps to a task URL — **to verify**).
- **Pagination:** `limit` max 50, `after` offset ≤ 9999, `hasMore` in meta. (`meta.total` looked like a narrow window — trust `hasMore`/`after`.)
- **Market reality (AU, digital):** saturated with WordPress/Shopify/SEO/automation tasks; budgets skew $10–$500 but real ones ($1k–$8k) exist.

Baseline snapshot of currently-open relevant tasks: see [`airtasker-baseline.json`](./airtasker-baseline.json).

---

## 7. CTA — next steps (for home)

**Nothing to build until we decide to. In priority order:**

1. **Verify the deep-link** — confirm the exact `airtasker.com` task-URL format from a `slug`, so "tap CRM lead → open in Airtasker app" is guaranteed.
2. **Write the lead-scoring rubric** — the exact formula that decides what becomes a CRM lead vs noise (freshness, offer count, budget floor, category, buyer rating).
3. **Design the CRM lead schema** — the fields a discovered task maps to in the Digital Urgency CRM.
4. **(Later, real build) Headless MCP client on Railway** running `browse_tasks` on a **cron tuned to AU peak posting hours** → insert leads into CRM → push to phone.
   - ⚠️ Engineering wrinkle: **headless OAuth token refresh** (the MCP expects interactive login).
   - ⚠️ Check **Airtasker ToS** on automated polling before shipping.
5. **Optional:** let Claude **draft the offer text** per lead so bidding is copy-paste in the app.

**Explicitly OUT of scope:** auto-submitting offers, replicating Airtasker's bidding/messaging, scraping, browser automation.

---

## 8. Repo contents

- `README.md` — this file.
- `airtasker-baseline.json` — baseline of open, relevant tasks + high-water-mark timestamp for change detection.

## 9. Push to GitHub (do this at home)

```bash
# create the empty repo "AirTasker" under github.com/WalidN1989 first, then:
git remote add origin https://github.com/WalidN1989/AirTasker.git
git branch -M main
git push -u origin main
```
