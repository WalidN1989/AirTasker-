# Airtasker MCP POC — Full Discussion Log

> A chronological record of the working session that produced this repo.
> **Date:** 2026-07-15 · **Participants:** Walid (Digital Urgency) + Claude Code (office Windows PC)
> Companion to `README.md`. This is the "how we got here" narrative; the README is the "what we concluded" reference.

---

## 0. The objective Walid set

Run a **simple proof-of-concept** to answer one question:

> *"If someone posts a task on Airtasker that matches our expertise, can Claude Code discover it through the **official** Airtasker MCP and identify it as a relevant opportunity?"*

Hard rules for the POC:
- Official Airtasker MCP only — **no scraping, no browser automation, no unofficial APIs, no invented tools.**
- **No** app / dashboard / database / scheduler / notifications / auto-offer submission.
- Confirm the **actual** tools the MCP exposes before trusting the docs.
- If a capability isn't supported, **say so** — don't build a workaround.
- Work **step by step**; don't run the whole experiment at once.

An expertise profile was provided (WordPress, Elementor, WooCommerce, Shopify, SEO, Local SEO, Google Business Profile, CRM, business/marketing automation, website fixes/migration, API/website integrations, etc.) with STRONG / POSSIBLE / NOT-RELEVANT matching tiers.

---

## 1. Phase 1 — Connect the MCP

**Inspected the environment first:**
- No Airtasker MCP configured anywhere (`~/.claude.json` had empty `mcpServers`; no project `.mcp.json`; the `AirTasker` folder was empty).
- The official docs page (`support.airtasker.com/.../58505062391321`) **403-blocked** the automated fetcher. Read it instead via the in-app browser (reading official docs — not scraping Airtasker).

**What the docs gave us:**
- Endpoint: `https://mcp.airtasker.com/mcp` · Transport **HTTP** · Auth **OAuth 2.0**.
- Claude Code setup: plugin (`/plugin marketplace add airtasker/airtasker-mcp-server` → `/plugin install airtasker@airtasker`) **or** `claude mcp add --transport http airtasker https://mcp.airtasker.com/mcp`, then `/mcp` to complete OAuth.
- Documented tools: `browse_tasks`, `get_task`, `post_task`, `get_my_tasks`, `get_offers`, `accept_offer`, `get_profile`.

**Reality on Walid's machine (Claude Code *desktop app*):**
- ❌ `/plugin` — *"not available in this environment"* (plugin system is terminal-CLI only).
- ❌ `claude` CLI — not installed / not on PATH (only the desktop app is present).
- Tried a project `.mcp.json` — the desktop app didn't pick it up (wrong mechanism for a remote OAuth server).
- ✅ **Solution that worked:** `Settings → Connectors → Add custom connector → URL https://mcp.airtasker.com/mcp → Connect → OAuth login in browser.` Walid completed the login. Tools then appeared to the agent automatically. (This is the same class of connector as Gmail/Google Drive already in the app.)

> Takeaway for the home Mac: the CLI plugin/`claude mcp add` path *will* work there; the Connectors panel is the desktop-app fallback.

---

## 2. Phase 2 — Verify the connection

**Confirmed exactly 7 tools** (matches the docs — nothing invented, nothing missing). In the connector, **all tools are set to "Needs approval"**, and the two write tools (`post_task`, `accept_offer`) are gated — good safety default.

**Ran a minimal read-only test browse** (10 most-recent tasks) → **live data returned**, newest task posted minutes earlier. Classified the sample against the expertise profile; correctly rejected false positives (e.g. "Web content proofreading" contains "web" but is a text job, not dev).

**Capabilities learned from the `browse_tasks` schema:**
- Sort by recency (`recent`, default), `price_asc`, `price_desc`.
- Geo filter: `lat` + `lon` + `radius_km`. Online/physical: `task_types` + `isOnline`.
- **No category filter** — use `search_term` free-text; results carry `categoryName`.
- Budget: `priceDollars` + `isFixedPrice` (often a placeholder — confirm with `get_task`).
- Competition read: `offersCount` per task.
- Reference: `slug` per task = its ID.
- Pagination: `limit` max 50, `after` ≤ 9999, `hasMore` in meta (trust `hasMore`, not `meta.total`).

---

## 3. Baseline search (experiment Step 2)

Ran 11 keyword `browse_tasks` searches across the high-priority services (`website`, `WordPress`, `Elementor`, `WooCommerce`, `Shopify`, `SEO`, `Google Business Profile`, `automation`, `CRM`, `landing page`, `API integration`), deduped, and saved a snapshot to `airtasker-baseline.json` (with a **high-water-mark timestamp** for later change detection).

**Findings:**
- The AU market is **saturated** with digital tasks in exactly our wheelhouse — matches appear constantly.
- **Heavy competition:** most open tasks already have **50–130 offers**.
- **Budgets skew low** ($10–$500 placeholders), but real high-value jobs exist — e.g. a **$8,000 "Custom Removalist Quote Calculator & CRM System Build"** with only 44 offers (a near-bullseye for the automation/CRM profile).
- **Freshly posted tasks have 0–5 offers** — this became the key strategic insight (see §5).

> Signal off: *"Baseline complete. Ready for the test task."* The blind discovery experiment (someone posts a task, Claude finds it without being told the title) is **ready but not yet run.**

---

## 4. The pivotal business-model discussion

Walid described the intended model: connect the MCP to the Digital Urgency **leads CRM**, run a **Railway cron** on AU peak hours, get leads on mobile, **approve a bid with his price**, and **chat with the client** — hoping to do it all without sitting at a PC.

**The critical correction (this reframed everything):**

The MCP is built for the **CUSTOMER** side, but Walid operates as a **SERVICE PROVIDER (Tasker) who bids to win jobs.** Of the 7 tools:
- ❌ **No `make_offer` / `submit_quote`** — you **cannot** place a bid via the MCP.
- ❌ **No messaging tool** — you **cannot** chat with a client via the MCP.
- `get_offers` / `accept_offer` are the **opposite direction** (you *posting* a job and *hiring* someone), not you winning work.
- `browse_tasks` is the **public, non-personalised feed** (same for everyone); OAuth only links the account-side tools.

**Conclusion, agreed:** the official MCP is a **discovery + qualification engine only.** The **actual bid and client chat must stay manual in the Airtasker app.** Replicating Airtasker's bidding/messaging is out of scope and unsupported.

Walid confirmed: *"I'm only a service provider… let the leads arrive in my CRM and then I click to open in the Airtasker app and take it forward."* ✅ Correct model.

---

## 5. The realistic architecture — a "lead radar," not an auto-bidder

The winning reframe: our edge is **speed**, not features. Fresh tasks (0–5 offers) are winnable; stale ones (50–130 offers) aren't. So the MCP's job is to catch matches **minutes after posting**.

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

**Principle:** discovery/qualification automated; **bid + conversation human; never auto-submit offers.**

Provider-side value of each read tool:
- `browse_tasks` = radar; `get_task` = full brief + let Claude draft the offer text; `get_profile` = vet the buyer (rating, tasks run); `firstPostedAt` + `offersCount` = score & prioritise.

**CTA / next steps (deferred to home):**
1. Verify the exact `airtasker.com` deep-link from a slug (so "tap → open in app" works).
2. Write the lead-scoring rubric (freshness, offer count, budget floor, category, buyer rating).
3. Design the CRM lead schema.
4. *(Later build)* Headless MCP client on Railway + cron on AU peak hours → insert leads → push to phone. Wrinkles: **headless OAuth token refresh**; check **Airtasker ToS** on automated polling.

---

## 6. Repo + housekeeping

- Created this repo locally (`git init`, README + baseline + `.gitignore` that blocks tokens/secrets), committed with a descriptive message.
- GitHub repo was created as **`AirTasker-`** (trailing hyphen).
- ⚠️ An automated safety layer **blocked the push** because the repo was **Public** and the README contained the **production CRM admin URL** + business plan. Correct catch.
- Walid **made the repo Private**, then **uploaded the files manually** (Git Credential Manager on the PC; no `gh` CLI available to the agent).
- Confirmed the substance lives in `README.md` §1–7 + `airtasker-baseline.json` — safe to delete the office copy; the Airtasker connector is tied to the account, not the PC.

---

## 7. One-line summary

> **The official Airtasker MCP works and reliably surfaces live, matching tasks — but only as a discovery radar. Walid is a service provider; the MCP cannot bid or message, so those stay manual in the Airtasker app. Next step is a lead engine that feeds scored opportunities into the Digital Urgency CRM. Nothing has been built yet — this POC proved feasibility and defined the realistic scope.**
