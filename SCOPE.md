# Super Growth — Raw Scope Document

**Version:** 0.1 (Pre-Lock)  
**Date:** 2026-07-18  
**Status:** DRAFT — subject to competitor analysis & review

---

## Module 1 — Lead Identification

### Features
- Multi-platform lead discovery (LinkedIn, Twitter/X, Instagram, Email)
- Search by keyword, industry, job title, company size, location
- Bulk lead import (CSV / existing CRM)
- Chrome extension for 1-click lead capture while browsing
- Real-time lead alerts (notify when someone matches saved criteria)
- Lead deduplication engine (same person across platforms = one record)

### Open Questions
- Which platforms do we hit at MVP vs later?
- Do we use official APIs or browser automation for LinkedIn/Instagram?

---

## Module 2 — Lead Qualification

### Features
- AI scoring engine (score leads 1–100 based on ICP match)
- ICP (Ideal Customer Profile) builder — define your perfect customer
- Filters: company size, funding stage, tech stack, hiring signals, intent data
- Auto-tagging (hot / warm / cold)
- Manual override — human can adjust AI score
- CRM-style pipeline view (Kanban or list)

### Open Questions
- What signals matter most for qualification? (intent data sources?)
- Do we build the scoring model or use a third-party intent data provider?

---

## Module 3 — Data Enrichment

### Features
- Firecrawl integration — scrape and map any company website
- Auto-extract: company description, products/services, team size, tech stack
- LinkedIn company page enrichment
- Email finder (Hunter.io-style)
- Phone number enrichment
- Social profile linking (connect LinkedIn + Twitter + website = one record)
- News & trigger alerts (funding rounds, new hires, product launches)

### Open Questions
- Build enrichment in-house or integrate Firecrawl / Clay / Clearbit APIs?
- How do we handle enrichment data freshness / staleness?

---

## Module 4 — Decision Maker Identification

### Features
- Org chart mapping from LinkedIn data
- Role-based filtering (CEO, CTO, Head of Growth, etc.)
- Buying committee identification (map all stakeholders at one company)
- Relationship scoring (how warm is our connection to each person?)
- "Find similar contacts" across other companies

### Open Questions
- How deep do we go on org charts at MVP?

---

## Module 5 — Personalized Outreach

### Features
- AI-generated personalised messages (email, DM, LinkedIn note)
- AI video generation / personalised video thumbnails
- Multi-channel sequences (e.g. LinkedIn DM → email → Twitter → SMS)
- A/B testing on messages
- Send-time optimisation
- Follow-up automation (if no reply after X days, send next step)
- Unsubscribe / opt-out management (compliance)
- Email warmup (to avoid spam folder)
- Analytics: open rate, reply rate, click rate, conversion per channel

### Open Questions
- Video outreach: integrate Loom / Sendspark / build in-house?
- Which email infrastructure? (SendGrid / Instantly / custom SMTP)

---

## Module 6 — Unified Inbox / Command Centre

### Features
- Single inbox for all replies (email, LinkedIn DMs, Twitter DMs, SMS)
- AI reply suggestions
- Conversation history per contact (across all platforms)
- Team collaboration — assign leads, leave notes, tag teammates
- Notification centre

---

## Module 7 — Analytics & Reporting

### Features
- Campaign performance dashboard
- Per-channel ROI breakdown
- Lead funnel (identified → qualified → enriched → contacted → replied → converted)
- Export reports (PDF / CSV)
- Team performance metrics

---

## Non-Functional Requirements

- **Compliance:** GDPR, CAN-SPAM, LinkedIn ToS — must be baked in from day one
- **Scale:** Should handle 10k+ leads per user without degrading
- **Integrations:** HubSpot, Salesforce, Notion, Slack (webhook support at minimum)
- **Security:** All contact data encrypted at rest and in transit

---

## MVP Scope (Proposed — to be locked after competitor analysis)

| Module | MVP | V2 |
|--------|-----|----|
| Lead Identification | LinkedIn + Email only | + Twitter, Instagram, SMS |
| Lead Qualification | AI scoring + manual pipeline | Intent data, advanced ICP |
| Data Enrichment | Firecrawl + email finder | Full enrichment suite |
| Decision Maker ID | Role-based filter | Org chart mapping |
| Outreach | Email sequences + LinkedIn DM | Video, SMS, multi-channel |
| Unified Inbox | Email + LinkedIn | All channels |
| Analytics | Basic campaign dashboard | Full funnel + team reporting |

---

## Competitor Analysis Status

- [ ] Apollo.io — pending screenshot review
- [ ] Clay — pending
- [ ] Instantly.ai — pending
- [ ] Lemlist — pending
- [ ] Expandi — pending
- [ ] PhantomBuster — pending
- [ ] Sales Robot — pending

*We will update this section as we go through each tool together.*
