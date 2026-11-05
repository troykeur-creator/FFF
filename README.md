# Friday Fraud Facts
### AI-Powered Weekly Fraud Intelligence Newsletter
**The Home Depot · Cybersecurity Fraud Prevention**

---

## Overview

Friday Fraud Facts is a weekly fraud intelligence newsletter system built for The Home Depot's Cybersecurity Fraud Prevention team. Claude AI researches live fraud news across six source categories, writes a polished newsletter in Troy Keur's voice, and delivers it to internal fraud partners every week.

**Live app:** [troykeur-creator.github.io/FFF](https://troykeur-creator.github.io/FFF)

---

## System Status

| Phase | Description | Status |
|-------|-------------|--------|
| Phase 1 | Manual generator — GitHub Pages | ✅ Live |
| Phase 2 | Automated Thursday draft → HD Outlook at noon CT | ✅ Live |
| Phase 3 | One-click Approve & Send to 56 recipients | 🔵 Pending web app deploy |
| Subscription Manager | Google Form + approval workflow | ✅ Live |

---

## Architecture

```
GitHub Pages (index.html)
        │
        ▼
fff-proxy.onrender.com  ← CORS proxy (Node/Express, Render.com free tier)
        │
        ▼
api.anthropic.com/v1/messages
        │
        ▼
Claude AI generates JSON newsletter
        │
        ▼
App renders HTML preview + email copy
```

**Cold-start handling:** The app pings the Render proxy to wake it before the main generation call. A 3-minute hard timeout prevents browser hangs if the server is unresponsive.

---

## Source Categories

The model searches all six categories every edition before writing.

| # | Category | Priority | Outlets |
|---|----------|----------|---------|
| 1 | Identity & Account Fraud | 🔴 Highest | Account openings, ATO, synthetic identity, credential stuffing, loyalty fraud |
| 2 | Peer Retailers | 🔴 High | Lowe's, Menards, Home Hardware, Walmart, Target, Costco, Best Buy, Wayfair, Amazon; Canadian Tire, Loblaw, Lowe's Canada, Walmart Canada |
| 3 | Retail E-Commerce | 🟡 Medium | RH-ISAC, Retail Dive, NRF, RILA, Pymnts.com, Retail Council of Canada |
| 4 | Fraud & Cyber Outlets | 🟡 Medium | KrebsOnSecurity, Dark Reading, BankInfoSecurity, SC Magazine, Bleeping Computer, The Record |
| 5 | Law Enforcement | 🟡 Medium | DOJ, FBI, FTC, Secret Service, RCMP, Canadian Anti-Fraud Centre, Competition Bureau Canada |
| 6 | Payments & Financial | 🟡 Medium | Visa, Mastercard, FTC, FinCEN, CFPB, Payments Canada, FCAC |

**Geographic scope:** United States primary. Canada included when stories involve a major Canadian retailer, a cross-border fraud trend, or a Canadian LE action with direct US implications.

**Story selection order:** Identity/ATO → Peer Retailer → E-Commerce → Broader landscape. 4–5 stories per edition spanning at least 3 categories.

---

## AI Model

Queries the Anthropic Models API at runtime and selects the best available model from this fallback chain:

```
claude-sonnet-5-5 → claude-opus-5-5 → claude-sonnet-4-6 → claude-haiku-5-5 → claude-haiku-4-5
```

Never breaks due to model deprecation.

---

## Newsletter Format

Every edition contains:

| Section | Description |
|---------|-------------|
| Orange header | HD branding, edition number, week date, team badge logo |
| Edition title | Unique catchy subtitle — style rotates every week |
| Opening note | Troy's take on the week — framing varies each edition |
| 4–5 story cards | Color-coded category badge, headline, 3–4 sentence summary, navy HD Relevance callout |
| This Week's Takeaway | Sharp insight + punchy bold closing line |
| Signature block | Troy Keur / Sr. Manager, Customer Identity Trust / The Home Depot |
| Sources & References | 4–8 verified outlets with direct article links where available |
| Footer | AI-drafted, reviewed by Cyber Fraud Prevention Team |

---

## Files

| File | Description |
|------|-------------|
| `index.html` | Standalone React 18 app — deployed to GitHub Pages |
| `FridayFraudFacts_Phase2.gs` | Google Apps Script — Thursday auto-draft to Outlook |
| `FridayFraudFacts_Phase3.gs` | Google Apps Script — Phase 3 Approve & Send (ready, pending deploy) |
| `FFF_SubscriptionManager.gs` | Google Apps Script — subscription form manager |
| `fff-proxy/server.js` | Node/Express CORS proxy — deployed to Render.com |
| `fff-proxy/package.json` | Proxy dependencies |
| `README.md` | This file |
| `DOCUMENTATION.md` | Full technical and operational documentation |

---

## Automation

### Phase 2 — Thursday Auto-Draft (Live)
- Script: `FridayFraudFacts_Phase2.gs`
- Platform: Google Apps Script (`script.google.com`)
- Trigger: Every Thursday at 12:00 PM CT
- Delivers draft to: `troy_a_keur1@homedepot.com`

### Phase 3 — Approve & Send (Pending)
- Script: `FridayFraudFacts_Phase3.gs` — built and tested
- Requires: Deploy as Web App in Google Apps Script
- See `DOCUMENTATION.md` → Phase 3 Deployment for step-by-step instructions

### Subscription Manager (Live)
- Form: [Subscribe to Friday Fraud Facts](https://docs.google.com/forms/d/e/1FAIpQLSeOp9jXyQFrFxwGcyu9DjXZ2s5xcbkeyPCwRX4dMmc_vpt4pA/viewform)
- Script: `FFF_SubscriptionManager.gs`
- Flow: Submit → Troy approves → welcome email sent → added to RECIPIENTS

---

## Recipient Management

Recipients are stored in the `RECIPIENTS` array at the top of `FridayFraudFacts_Phase3.gs`.

- **Add:** Append `"firstname_lastname@homedepot.com"` to the array
- **Remove:** Delete that line from the array
- **New requests:** Come through the subscription form — Troy receives approval email automatically

Current list: **56 internal HD recipients**

---

## Technology Stack

| Component | Technology |
|-----------|------------|
| Frontend | React 18 + Babel Standalone (single HTML file, no build step) |
| AI | Claude (Anthropic) — model auto-selected at runtime |
| CORS Proxy | Node.js / Express on Render.com free tier |
| Storage | Browser localStorage (edition number, 52-edition history, used topics) |
| Automation | Google Apps Script |
| Hosting | GitHub Pages |
| Estimated cost | ~$0.25–$0.50 per generation (~$1–2/month) |

---

## Privacy & Security

- No user data stored on external servers
- Newsletter content generated fresh at generation time — not cached anywhere
- Anthropic API key stored in Google Apps Script environment variables only — never in the HTML or repo
- No HD customer data, proprietary data, or internal systems accessed
- All AI-generated content reviewed by Troy Keur before distribution
- Subscription form enforces `@homedepot.com` email addresses
- Internal use only

---

## Support & Changes

All changes, troubleshooting, and enhancements are handled through the **Friday Fraud Facts session in Claude.ai**. Full build history and context are preserved across sessions. Examples of things you can ask:

- *"The copy button stopped working"*
- *"Add a new search source category"*
- *"Ready to deploy Phase 3 — walk me through it"*
- *"Update the recipient list"*
- *"Reset topic memory starting at edition 42"*
- *"The proxy is timing out again"*

---

*Friday Fraud Facts · The Home Depot Cybersecurity Fraud Prevention · Internal Use Only*
