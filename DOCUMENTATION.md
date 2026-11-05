# Friday Fraud Facts — Technical & Operational Documentation

**The Home Depot · Cybersecurity Fraud Prevention**
**Maintained by:** Troy Keur, Sr. Manager, Customer Identity Trust
**Last updated:** November 2026

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architecture](#2-architecture)
3. [index.html — The Generator App](#3-indexhtml--the-generator-app)
4. [CORS Proxy (fff-proxy)](#4-cors-proxy-fff-proxy)
5. [System Prompt & AI Instructions](#5-system-prompt--ai-instructions)
6. [Phase 2 — Thursday Auto-Draft](#6-phase-2--thursday-auto-draft)
7. [Phase 3 — Approve & Send (Pending)](#7-phase-3--approve--send-pending)
8. [Subscription Manager](#8-subscription-manager)
9. [Deploying to GitHub Pages](#9-deploying-to-github-pages)
10. [Deploying the Proxy to Render.com](#10-deploying-the-proxy-to-rendercom)
11. [Troubleshooting](#11-troubleshooting)
12. [Making Changes](#12-making-changes)

---

## 1. System Overview

Friday Fraud Facts is a weekly fraud intelligence newsletter delivered to 56 internal Home Depot fraud and security partners every Friday. The system has three phases:

| Phase | What it does | Status |
|-------|-------------|--------|
| **Phase 1** | Manual generator at troykeur-creator.github.io/FFF — Troy clicks Generate, reviews the output, and uses Copy as Email to paste into Outlook | ✅ Live |
| **Phase 2** | Google Apps Script automatically generates a draft every Thursday at noon CT and emails it to Troy's HD inbox | ✅ Live |
| **Phase 3** | Troy clicks Approve in his inbox → the newsletter sends to all 56 recipients automatically | 🔵 Built, pending deploy |

The generator, proxy, and automation scripts are all independent — any one of them can be updated without touching the others.

---

## 2. Architecture

### Component Map

```
┌─────────────────────────────────────────────────────────┐
│                   GitHub Pages                          │
│              troykeur-creator.github.io/FFF             │
│                                                         │
│  index.html                                             │
│  ├── React 18 (Babel Standalone — no build step)        │
│  ├── SYSTEM_PROMPT  — AI instructions + JSON schema     │
│  ├── generate()     — orchestrates the full flow        │
│  │   ├── Wake ping → fff-proxy.onrender.com/            │
│  │   ├── resolveModel() → /api/models                   │
│  │   └── fetch() → /api/claude  (3-min timeout)         │
│  ├── buildEmailHtml() — MSO-compatible email HTML       │
│  └── savePdf()        — printable PDF via browser print │
└──────────────────────┬──────────────────────────────────┘
                       │  HTTPS POST
                       ▼
┌─────────────────────────────────────────────────────────┐
│              Render.com (free tier)                     │
│               fff-proxy.onrender.com                    │
│                                                         │
│  server.js (Node/Express)                               │
│  ├── GET  /          — health check / wake endpoint     │
│  ├── GET  /api/models — proxies Anthropic models list   │
│  └── POST /api/claude — strips web_search tool,        │
│         forwards to api.anthropic.com/v1/messages       │
│         AbortSignal: 120s to Anthropic                  │
│         server.timeout: 300s                            │
└──────────────────────┬──────────────────────────────────┘
                       │  HTTPS POST (x-api-key header)
                       ▼
              api.anthropic.com/v1/messages
              (Anthropic Claude API)
```

### Why the Proxy Exists

GitHub Pages serves static files. The browser cannot call `api.anthropic.com` directly because:
- The Anthropic API requires an `x-api-key` header — exposing the key in browser JavaScript would make it public
- Anthropic does not allow cross-origin requests directly from browsers (CORS)

The proxy holds the API key in a server-side environment variable, adds it to every request, and forwards the response back.

### Why the Proxy Strips the `web_search` Tool

An earlier version of `index.html` included `tools: [{ type: "web_search_20250305" }]` in the request body. When a tool is included, Claude returns `stop_reason: "tool_use"` and the app's single-turn JSON parser fails with "Could not find JSON in response." The proxy strips any `tools` parameter as a safety net; `index.html` no longer sends one.

---

## 3. index.html — The Generator App

### Key Sections

| Section | Lines (approx.) | Description |
|---------|-----------------|-------------|
| `APP_LOGO` | ~15 | Base64-encoded HD badge logo |
| `categories` | ~190 | Color map for story category badges |
| `SYSTEM_PROMPT` | ~20–190 | Full AI instructions, JSON schema, source rules |
| `resolveModel()` | ~232–273 | Calls `/api/models`, picks best available from fallback list |
| `generate()` | ~275–445 | Main function: wake ping → model → fetch → parse → save |
| `cleanJson()` | ~329–380 | Multi-pass JSON sanitizer (unicode, unescaped chars) |
| `buildEmailHtml()` | ~448–518 | Builds MSO-compatible HTML email string |
| `savePdf()` | ~519–550 | Opens print-ready HTML in new tab |
| `copyEmail()` | ~553–580 | Copies email HTML to clipboard, sets `copied` feedback |
| React render | ~600–740 | UI: controls bar, spinner, error, content, history modal, roadmap |

### Model Fallback Chain

At generation time, `resolveModel()` fetches the live model list from `/api/models` and picks the first match from:

```
claude-sonnet-5-5  (preferred)
claude-opus-5-5
claude-sonnet-4-6  (always available — safe fallback)
claude-haiku-5-5
claude-haiku-4-5   (last resort)
```

If the models endpoint is unreachable (10-second timeout), it defaults to `claude-sonnet-5-5`.

### Cold-Start Flow

Render.com free-tier instances sleep after ~15 minutes of inactivity. The startup sequence inside `generate()` is:

1. **Wake ping** — `GET https://fff-proxy.onrender.com/` with 35s timeout. If it fails, continues anyway.
2. **Model check** — `GET /api/models` with 10s timeout. Falls back silently.
3. **Main generation** — `POST /api/claude` with **3-minute hard timeout** (`AbortSignal.timeout(180000)`).

If the 3-minute timeout fires, the user sees: *"The server took too long to respond. It may still be warming up — please wait 30 seconds and try again."*

### localStorage Keys

| Key | Contents | Retained |
|-----|----------|----------|
| `fff_edition_num` | Current edition number string | Indefinitely |
| `fff_used_topics` | Array of up to 8 recent edition headline strings | Rolling — oldest dropped when >8 |
| `fff_history` | Array of up to 52 edition objects `{editionNum, weekLabel, date, content}` | Rolling — oldest dropped when >52 |

### JSON Sanitizer

The model sometimes returns special Unicode characters that break `JSON.parse()`. `cleanJson()` runs three passes:

1. **Unicode substitution** — smart quotes → `"`, smart apostrophes + acute accent + backtick → `'`, em/en dash → `-`, ellipsis → `...`, zero-width spaces → ` `
2. **Unescaped control character fix** — walks character by character, escaping bare newlines and tabs inside JSON strings
3. **Parse attempts** — `JSON.parse()` → `JSON.parse(cleanJson())` → `Function("return (" + json + ")()")` (lenient evaluator as last resort)

If all three fail, the error message shows the offending line to help diagnose.

---

## 4. CORS Proxy (fff-proxy)

### Location

- **Repo path:** `fff-proxy/`
- **Files:** `server.js`, `package.json`
- **Deployed at:** `https://fff-proxy.onrender.com`

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/` | Health check — returns `{"status": "Friday Fraud Facts Proxy — OK"}` |
| `GET` | `/api/models` | Proxies `GET api.anthropic.com/v1/models` — 15s timeout |
| `POST` | `/api/claude` | Proxies `POST api.anthropic.com/v1/messages` — strips `tools`, 120s Anthropic timeout, 300s server timeout |

### Environment Variable

| Variable | Value |
|----------|-------|
| `ANTHROPIC_API_KEY` | Anthropic API key — set in Render.com Dashboard → Environment |

### Render.com Free Tier Behavior

- Sleeps after ~15 minutes of inactivity
- Cold start takes 20–35 seconds
- The wake ping in `index.html` handles this automatically
- **No monthly cost** at current usage levels (~4 requests/month)

---

## 5. System Prompt & AI Instructions

The `SYSTEM_PROMPT` constant in `index.html` contains everything the model needs. Key sections:

### Voice & Persona
Troy Keur writes as a practitioner-expert who reads widely, synthesizes fast, and speaks plainly. The newsletter is for a team, not the public. The voice is authoritative but human — not corporate.

### JSON Output Schema

The model must return exactly this structure (no prose before or after the JSON):

```json
{
  "edition_title": "Unique catchy subtitle — rotates style every week",
  "opening": "2–3 sentences — Troy's take on the week",
  "stories": [
    {
      "headline": "sharp internal headline",
      "category": "one of the 8 defined categories",
      "summary": "3–4 sentences — what happened, why it matters, peer context",
      "relevance": "1 sentence connecting to Home Depot's fraud program"
    }
  ],
  "takeaway": "2 sentences of sharp insight + 1 witty closing line",
  "sign_off": "One bold, punchy closing line — no name",
  "sources": [
    {
      "name": "KrebsOnSecurity",
      "url": "https://krebsonsecurity.com",
      "article_url": "https://krebsonsecurity.com/2026/11/specific-article/"
    }
  ]
}
```

### Story Categories (8 defined)

`Retail Fraud` · `E-Commerce` · `Account Takeover` · `Returns Abuse` · `Organized Retail Crime` · `Payments & Cards` · `AI & Fraud Tech` · `Regulatory`

### Sources Schema

Each source object requires `name` and `url` (homepage). `article_url` is optional but strongly preferred — links directly to the specific cited article. In the app UI, the source name links to `article_url` if present, with a small `(site)` link to the homepage beside it.

### Approved Source List

**US Cyber/Fraud:** KrebsOnSecurity, Dark Reading, BankInfoSecurity, SC Magazine, Bleeping Computer, The Record (Recorded Future News), SecurityWeek, Threatpost

**US Retail/Payments:** Retail Dive, NRF, Pymnts.com, RILA, Digital Commerce 360

**US Government/LE:** FBI IC3, FTC, DOJ, Secret Service, FinCEN, CFPB

**Payment Networks:** Visa, Mastercard

**Canadian:** RCMP, Canadian Anti-Fraud Centre, Competition Bureau Canada, Financial Consumer Agency of Canada, Retail Council of Canada

The model is explicitly instructed not to invent sources outside this list.

### Deduplication

The last 8 edition headline strings are passed in the user message. The model is instructed not to repeat topics, company names, or fraud types from those editions, and to vary the opening paragraph style and edition title structure.

### Legal & PR Guardrails

- Report what was alleged or charged — never state guilt
- Do not name specific HD customers, employees, or internal systems
- Do not reproduce full article text (fair use)
- Stories must be from the past 7 days

---

## 6. Phase 2 — Thursday Auto-Draft

### What It Does

Every Thursday at 12:00 PM CT, a Google Apps Script calls the same CORS proxy with the same system prompt, generates a newsletter, and emails the HTML draft to `troy_a_keur1@homedepot.com`.

### File

`FridayFraudFacts_Phase2.gs` — deployed in Google Apps Script at `script.google.com`

### Setup

1. Open [script.google.com](https://script.google.com)
2. Open the `FridayFraudFacts_Phase2` project
3. The trigger is set to run every Thursday at 12:00 PM CT (verify under **Triggers**)
4. The API key is stored under **Project Settings → Script Properties** as `ANTHROPIC_API_KEY`

### If the Draft Doesn't Arrive

1. Check the script's execution log: `script.google.com` → **My Projects** → open project → **Executions**
2. Common cause: Render.com cold start — the script does not include a wake ping (the proxy timeout in GAS is longer than the browser)
3. If the execution failed, manually run `generateAndEmail()` from the script editor

---

## 7. Phase 3 — Approve & Send (Pending)

### What It Does

Troy receives the Thursday draft in Outlook. The email contains an **Approve & Send** button. Clicking it triggers a Google Apps Script Web App that sends the finalized newsletter to all 56 recipients.

### File

`FridayFraudFacts_Phase3.gs` — built and tested, **not yet deployed as a Web App**.

### Deployment Steps

1. Open [script.google.com](https://script.google.com) and open `FridayFraudFacts_Phase3`
2. Click **Deploy → New deployment**
3. Select type: **Web app**
4. Set **Execute as:** Me (troy_a_keur1@homedepot.com)
5. Set **Who has access:** Anyone with the link (or Anyone within Home Depot if the org allows)
6. Click **Deploy** — copy the Web App URL
7. Paste the Web App URL into the `APPROVE_URL` constant in `FridayFraudFacts_Phase2.gs`
8. The Thursday draft email will now include an Approve button that points to that URL

### Recipient List

The `RECIPIENTS` array is at the top of `FridayFraudFacts_Phase3.gs`. Each entry is a `@homedepot.com` email address. To update:

- **Add:** Append `"firstname_lastname@homedepot.com",` to the array
- **Remove:** Delete that line
- **Current count:** 56 recipients

---

## 8. Subscription Manager

### What It Does

Subscribers fill out a Google Form. The form response triggers a Google Apps Script that emails Troy for approval. When Troy approves, the subscriber gets a welcome email and is added to the RECIPIENTS list.

### File

`FFF_SubscriptionManager.gs` — deployed in Google Apps Script

### Form Link

[Subscribe to Friday Fraud Facts](https://docs.google.com/forms/d/e/1FAIpQLSeOp9jXyQFrFxwGcyu9DjXZ2s5xcbkeyPCwRX4dMmc_vpt4pA/viewform)

### Form enforces `@homedepot.com` — no external email addresses are accepted.

---

## 9. Deploying to GitHub Pages

The app is a single `index.html` file. No build step. No dependencies to install.

**To deploy an update:**

1. Download the new `index.html` from Claude.ai
2. Go to [github.com/troykeur-creator/FFF](https://github.com/troykeur-creator/FFF)
3. Click `index.html` → click the **pencil icon** (Edit)
4. Select all → delete → paste the new file contents
5. Click **Commit changes**
6. GitHub Pages redeploys automatically in ~60 seconds
7. Verify at [troykeur-creator.github.io/FFF](https://troykeur-creator.github.io/FFF)

**Alternatively:** Upload the file directly under **Add file → Upload files**.

---

## 10. Deploying the Proxy to Render.com

The proxy only needs redeployment if `server.js` changes.

**To deploy a proxy update:**

1. Download the new `server.js` from Claude.ai
2. Go to [render.com](https://render.com) → open the `fff-proxy` service
3. Under **Settings**, the repo is linked to GitHub — push `server.js` to the `fff-proxy/` folder in the repo and Render auto-deploys
4. **OR:** Go to **Deploys** → **Manual Deploy** → **Deploy latest commit** if the file is already in the repo

**To check proxy health:**

Visit `https://fff-proxy.onrender.com/` — should return `{"status":"Friday Fraud Facts Proxy — OK"}`. If it hangs for 20–30 seconds then returns, that's a normal cold start.

**Environment variable (do not lose this):**

| Variable | Where to set |
|----------|-------------|
| `ANTHROPIC_API_KEY` | Render Dashboard → fff-proxy service → **Environment** tab |

---

## 11. Troubleshooting

### "Could not find JSON in response"

The model returned a non-JSON response (tool use loop, refusal, or malformed output).

- **Check:** Did the proxy deploy correctly? Visit `https://fff-proxy.onrender.com/` and confirm it returns the health check JSON.
- **Check:** Open browser DevTools → Network tab → look at the `/api/claude` response body. If `stop_reason` is `"tool_use"`, the `tools` parameter is somehow back in the request.
- **Fix:** Hit **Regenerate** — transient model behavior clears on retry.

### "The server took too long to respond"

The Render.com free tier cold start exceeded 3 minutes (rare — usually takes 20–35s).

- **Fix:** Wait 30–60 seconds and hit **Generate** again. The server is now warm.
- **If it keeps happening:** Check Render.com dashboard for service errors under **Logs**.

### "Parse error near: }"

The model returned valid JSON that couldn't be parsed after sanitization.

- **Fix:** Hit **Regenerate** — the model occasionally produces malformed JSON on the first attempt.
- **If persistent:** The system prompt schema may have been corrupted in a recent edit. Compare `SYSTEM_PROMPT` in `index.html` against the schema in Section 5 of this document.

### Sources Not Showing

- **Check:** Was the edition generated with the current version of `index.html`? Old editions in History were saved without sources.
- **Check:** Open DevTools → Console. Look for a JSON parse error on the sources array.
- **Fix:** Regenerate — the sources schema is now required in the prompt.

### History Is Empty / Lost

localStorage is per-browser and per-origin. If you open the app in a different browser or clear browser data, history is gone.

- **This is expected behavior** — history is a convenience, not a system of record.
- Keep important editions by using **Copy as Email** and saving the draft in Outlook.

### Thursday Draft Didn't Arrive

1. Check [script.google.com](https://script.google.com) → open `FridayFraudFacts_Phase2` → **Executions** tab
2. If execution failed: click it to see the error, then run `generateAndEmail()` manually from the editor
3. Common causes: Render cold start (proxy timeout), Anthropic API key expired, quota exceeded

---

## 12. Making Changes

All changes to this system are made through the **Friday Fraud Facts session in Claude.ai**. The session has full context of every change ever made to every file.

### How to Request a Change

Just describe what you want in plain language. Examples:

| Request | What gets updated |
|---------|------------------|
| "Add a new outlet to the source list" | `index.html` SYSTEM_PROMPT + approved source list |
| "Change the edition number to 32" | Nothing to change — edition number is a text input in the app |
| "The copy button isn't working" | `index.html` `copyEmail()` function |
| "Add a new fraud category" | `index.html` `categories` map + SYSTEM_PROMPT category list |
| "Deploy Phase 3" | Walk through Section 7 of this document |
| "Update the recipient list" | `FridayFraudFacts_Phase3.gs` RECIPIENTS array |
| "The proxy is timing out" | `fff-proxy/server.js` timeout values |
| "Reset deduplication memory" | Clear `fff_used_topics` from browser DevTools → Application → localStorage |

### What Gets Deployed Where

| Changed file | Deploy to |
|-------------|-----------|
| `index.html` | GitHub Pages (Section 9) |
| `fff-proxy/server.js` | Render.com (Section 10) |
| `FridayFraudFacts_Phase2.gs` | Google Apps Script (paste into script editor, save) |
| `FridayFraudFacts_Phase3.gs` | Google Apps Script (paste into script editor, save + redeploy Web App) |
| `FFF_SubscriptionManager.gs` | Google Apps Script (paste into script editor, save) |

---

*Friday Fraud Facts · The Home Depot Cybersecurity Fraud Prevention · Internal Use Only*
*Document maintained in the Friday Fraud Facts Claude.ai session — update this file whenever the system changes.*
