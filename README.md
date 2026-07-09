# Friday Fraud Facts
### AI-Powered Weekly Fraud Intelligence Newsletter
**The Home Depot · Cybersecurity Fraud Prevention**

---

## Overview

Friday Fraud Facts is a fully automated weekly fraud intelligence newsletter system built for The Home Depot's Cybersecurity Fraud Prevention team. It uses Claude AI to research live fraud news, write a polished newsletter in Troy Keur's voice, and deliver it to internal fraud partners every week.

**Live app:** [troykeur-creator.github.io/FFF](https://troykeur-creator.github.io/FFF)

---

## System Status

| Phase | Description | Status |
|-------|-------------|--------|
| Phase 1 | Manual generator — Claude.ai + GitHub Pages | ✅ Live |
| Phase 2 | Automated Thursday draft → HD Outlook at noon CT | ✅ Live |
| Phase 3 | One-click Approve & Send to 56 recipients | 🔵 Pending web app deploy |
| Subscription Manager | Google Form + approval workflow | ✅ Live |

---

## How It Works

```
Every Thursday at noon CT
        ↓
Google Apps Script
        ↓
Claude AI (claude-sonnet-5 with web search)
        ↓
Searches 6 fraud intelligence source categories
        ↓
Generates branded HTML newsletter in Troy's voice
        ↓
Emails draft to troy_a_keur1@homedepot.com
        ↓
Troy reviews → [Phase 3: click Approve] → sends to 56 recipients
```

---

## Source Categories (searched every edition)

| # | Category | Sources |
|---|----------|---------|
| 1 | Identity & ATO *(highest priority)* | Fraudulent account openings, ATO, synthetic identity, credential stuffing, loyalty fraud |
| 2 | Peer Retailers *(high priority)* | Lowe's, Menards, Walmart, Target, Costco, Best Buy, Wayfair, Amazon |
| 3 | Retail E-Commerce | RH-ISAC, Retail Dive, NRF, RILA, Pymnts.com |
| 4 | Fraud & Cyber Outlets | KrebsOnSecurity, Dark Reading, BankInfoSecurity, SC Magazine |
| 5 | Law Enforcement | DOJ, FBI, FTC, Secret Service press releases |
| 6 | Payments & Financial | Visa, Mastercard, FTC, FinCEN, CFPB |

**Priority rules:** Identity/ATO → Peer Retailers → E-Commerce → Broader landscape. US-only. Peer retailer wins tiebreaker.

---

## Email Format

Every edition contains:
- **Orange header** — HD branding, edition number, week date, team badge logo
- **Edition title block** — unique catchy subtitle each week
- **Opening note** — Troy's take on the week (varies in style every edition)
- **4–5 story cards** — color-coded category badge, headline, summary, navy HD Relevance callout
- **This Week's Takeaway** — sharp closing insight + punchy sign-off line
- **Signature block** — Troy Keur / Sr. Manager, Customer Identity Trust / The Home Depot
- **Footer** — Drafted with AI assistance and reviewed by Cyber Fraud Prevention Team

---

## AI Model

Uses `claude-sonnet-5` with automatic fallback chain:
```
claude-sonnet-5 → claude-sonnet-4-6 → claude-sonnet-4-5 → claude-opus-4-8 → claude-haiku-4-5
```
Queries the live Anthropic Models API at runtime — automatically selects the best available model. Never breaks due to model deprecation.

---

## Files

| File | Description |
|------|-------------|
| `index.html` | Standalone app — deployed to GitHub Pages |
| `FridayFraudFacts.jsx` | React source — used in Claude.ai artifact |
| `FridayFraudFacts_Phase2.gs` | Google Apps Script — Thursday auto-draft |
| `FridayFraudFacts_Phase3.gs` | Google Apps Script — Phase 3 with Approve & Send |
| `FFF_SubscriptionManager.gs` | Google Apps Script — form subscription manager |

---

## Automation Setup

### Phase 2 (Live)
- Script: `FridayFraudFacts_Phase2.gs`
- Platform: Google Apps Script (`script.google.com`)
- Trigger: Every Thursday at 12:00 PM CT
- Delivers to: `troy_a_keur1@homedepot.com`

### Subscription Form (Live)
- Form: [Subscribe to Friday Fraud Facts](https://docs.google.com/forms/d/e/1FAIpQLSeOp9jXyQFrFxwGcyu9DjXZ2s5xcbkeyPCwRX4dMmc_vpt4pA/viewform)
- Script: `FFF_SubscriptionManager.gs`
- Flow: Submit → Troy approves → welcome email sent → added to RECIPIENTS

### Phase 3 (Pending)
- Script: `FridayFraudFacts_Phase3.gs` — built and ready
- Needs: Deploy as Web App in Google Apps Script
- See user guide for deployment steps

---

## Recipient Management

Recipients are stored in the `RECIPIENTS` array at the top of `FridayFraudFacts_Phase3.gs`.

- **Add someone:** append their `@homedepot.com` email to the array
- **Remove someone:** delete their line from the array
- **New requests:** managed via the subscription form — approval emails sent to Troy automatically

Current list: **56 internal HD recipients**

---

## Technology Stack

| Component | Technology |
|-----------|-----------|
| Frontend | React 18 + Babel Standalone (single HTML file) |
| AI Model | Claude Sonnet 5 (Anthropic) |
| Web Search | Anthropic web_search tool (live at generation time) |
| Email Format | Table-based HTML with MSO conditional comments |
| Storage | Browser localStorage (edition number + 8-edition history) |
| Automation | Google Apps Script |
| Hosting | GitHub Pages |
| Cost | ~$0.25–$0.50 per generation (~$1–2/month) |

---

## Privacy & Security

- No user data stored on external servers
- Content generated fresh each week via direct API call
- Anthropic API key stored in Google Apps Script environment variables only
- No HD customer data, proprietary data, or internal systems accessed
- AI content reviewed by Troy Keur before distribution
- Internal HD only — `@homedepot.com` emails enforced on subscription form

---

## Support

All changes and troubleshooting handled via the **Friday Fraud Facts conversation in Claude.ai**. Full build history and context preserved. Examples:
- *"The copy button stopped working"*
- *"Add a new search source"*
- *"Ready to deploy Phase 3"*
- *"Update the recipient list"*
- *"Reset topic memory to edition 42"*

---

*Friday Fraud Facts · The Home Depot Cybersecurity Fraud Prevention · Internal Use Only*
