# LinkedIn Scraping Strategy — Production Architecture
**Date:** 2026-07-18  
**Status:** Approved approach post-discovery session  
**Goal:** Scrape thousands of LinkedIn leads by keyword reliably, without bans, at scale.

---

## What We Know (Post-Discovery)

| Method | Status | Notes |
|--------|--------|-------|
| Voyager `/search/blended` REST API | ❌ DEAD | 404 — LinkedIn migrated to RSC/SDUI |
| RSC `/flagship-web/rsc-action` endpoints | ⚠️ RISKY | POST-based, opaque sduiid, hard to maintain |
| Browser navigation + HTML parsing | ✅ WORKS | Confirmed in Session 001. This is the approach. |
| Voyager profile/connection APIs | ✅ PARTIAL | Still alive for non-search operations |

---

## The Endpoints We Can Hit

### Via Browser Navigation (Navigate → Wait → Parse HTML)

| Endpoint | URL Pattern | Data Available |
|----------|-------------|---------------|
| People Search | `/search/results/people/?keywords={q}&origin=GLOBAL_SEARCH_HEADER` | Name, title, company, location, connection degree, follower count |
| People Search + Filters | `?keywords={q}&geoUrn={location}&currentCompany={id}` | Geo + company filtered results |
| Company Search | `/search/results/companies/?keywords={q}` | Company name, industry, size, location |
| Individual Profile | `/in/{profile_slug}/` | Full profile: name, headline, summary, experience, education, skills |
| Company Page | `/company/{slug}/` | Company description, size, industry, website, follower count |
| Company Employees | `/company/{slug}/people/` | Employee list with titles and locations |
| Job Listings | `/jobs/search/?keywords={q}&location={loc}` | Job title, company, location, date posted |
| Sales Navigator | `/sales/search/people` | Deeper filters (requires Sales Nav subscription) |

### Via Voyager API (Still Alive — For Enrichment)

| Endpoint | URL Pattern | Data Available |
|----------|-------------|---------------|
| Profile Lookup | `/voyager/api/identity/profiles/{id}` | Full structured profile JSON |
| Connection Degree | `/voyager/api/relationships/connectionOf` | 1st/2nd/3rd degree |
| Messaging | `/voyager/api/messaging/conversations` | Send DMs (with auth) |
| Company Details | `/voyager/api/organization/companies?q=universalName&universalName={slug}` | Company structured data |

---

## The Stack

### Layer 1 — Browser Engine
**Tool:** Playwright (Python) + `playwright-stealth`  
**Why:** Playwright supports persistent browser contexts (save session to disk). `playwright-stealth` patches ~20 browser fingerprint properties that expose automation (navigator.webdriver, plugins list, WebGL renderer, etc.)

```python
pip install playwright playwright-stealth
playwright install chromium
```

### Layer 2 — Persistent Sessions (Per Account)
Each LinkedIn account gets its own browser profile saved to disk. Log in once → reuse forever.

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    # Each account gets its own user_data_dir
    context = p.chromium.launch_persistent_context(
        user_data_dir=f"./sessions/account_{account_id}",
        headless=True,
        args=["--disable-blink-features=AutomationControlled"]
    )
    page = context.new_page()
    # First run: log in manually, session is saved
    # Every subsequent run: already authenticated
```

### Layer 3 — Anti-Detection Rules
NEVER skip these. Each one is a separate detection vector LinkedIn checks:

1. **Stealth patch** — apply before any page load
   ```python
   from playwright_stealth import stealth_sync
   stealth_sync(page)
   ```

2. **Random delays** — between every action, not just between pages
   ```python
   import random, time
   time.sleep(random.uniform(3.0, 8.0))  # between page loads
   time.sleep(random.uniform(0.5, 2.0))  # between scroll events
   ```

3. **Human scroll pattern** — scroll before reading, not instantly
   ```python
   page.evaluate("window.scrollTo(0, document.body.scrollHeight * 0.3)")
   time.sleep(random.uniform(1, 2))
   page.evaluate("window.scrollTo(0, document.body.scrollHeight * 0.7)")
   time.sleep(random.uniform(1, 2))
   ```

4. **Realistic viewport** — never use default headless dimensions
   ```python
   context = p.chromium.launch_persistent_context(
       viewport={"width": 1440, "height": 900},
       user_agent="Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 ..."
   )
   ```

5. **Never exceed 80–100 actions per account per day** — hard limit

6. **Randomize the schedule** — spread actions across the day, not in bursts

### Layer 4 — Account Pool Manager
A simple pool that tracks each account's daily action count and rotates when approaching the limit.

```python
class AccountPool:
    def __init__(self, accounts):
        self.accounts = {acc_id: {"used": 0, "limit": 80} for acc_id in accounts}
    
    def get_available(self):
        for acc_id, stats in self.accounts.items():
            if stats["used"] < stats["limit"]:
                return acc_id
        return None  # all accounts at limit — wait until tomorrow
    
    def log_action(self, acc_id):
        self.accounts[acc_id]["used"] += 1
    
    def reset_daily(self):
        for acc_id in self.accounts:
            self.accounts[acc_id]["used"] = 0
```

### Layer 5 — Proxy Layer (For Scale)
- **Under 500 leads/day:** Not needed. Use real IPs from your accounts.
- **500–5000 leads/day:** Residential proxies. Each account gets one dedicated IP.
- **5000+ leads/day:** Rotate residential proxy pool. Recommended providers: Brightdata, Oxylabs, Smartproxy.

```python
# Assign one residential IP per account — looks like one person browsing from one location
context = p.chromium.launch_persistent_context(
    proxy={"server": f"http://{account_proxy_ip}:{port}",
           "username": proxy_user, "password": proxy_pass}
)
```

### Layer 6 — HTML Parser
After page load, extract structured data from the rendered HTML:

```python
def parse_people_results(page_text):
    leads = []
    lines = [l.strip() for l in page_text.split("\n") if len(l.strip()) > 2]
    
    i = 0
    while i < len(lines):
        line = lines[i]
        
        # Named result: "John Smith • 2nd"
        import re
        match = re.match(r"^(.+?)\s+[•·]\s+(1st|2nd|3rd\+)$", line)
        if match:
            leads.append({
                "name": match.group(1).strip(),
                "degree": match.group(2),
                "title": lines[i+1] if i+1 < len(lines) else "",
                "location": lines[i+2] if i+2 < len(lines) else "",
            })
            i += 3
        
        # Hidden result: "LinkedIn Member"
        elif line == "LinkedIn Member" and i+1 < len(lines):
            leads.append({
                "name": None,
                "degree": "3rd+",
                "title": lines[i+1],
                "location": lines[i+2] if i+2 < len(lines) else "",
            })
            i += 3
        else:
            i += 1
    
    return leads
```

### Layer 7 — Data Pipeline
```
Search → Parse → Deduplicate → Enrich (Firecrawl/Voyager) → Store
```

- **Deduplication:** Hash on (name + company) or LinkedIn profile URL slug
- **Storage:** PostgreSQL with a `leads` table. Profile URL slug is the unique key.
- **Enrichment queue:** After identification, push profile slugs to enrichment queue → load individual profile page → extract full details

---

## What You Can Extract Per Lead (Full Profile Scrape)

From **search results page:**
- Name (if 1st/2nd degree)
- Headline / current title
- Current company
- Location
- Connection degree
- Follower count

From **individual profile page** (`/in/{slug}/`):
- Full name
- About / summary
- All experience entries (company, title, dates, description)
- Education
- Skills
- Certifications
- Profile URL slug

From **company page** (`/company/{slug}/people/`):
- All employees with titles and departments
- Company HQ, size, industry, website

---

## Scale Estimate

| Accounts in Pool | Actions/Day Each | Leads/Day | Leads/Month |
|-----------------|-----------------|-----------|-------------|
| 5 accounts | 80 | ~400 | ~12,000 |
| 10 accounts | 80 | ~800 | ~24,000 |
| 20 accounts | 80 | ~1,600 | ~48,000 |
| 50 accounts | 80 | ~4,000 | ~120,000 |

---

## Risk Tiers

| Risk | Mitigation |
|------|-----------|
| Account ban | Hard daily limits + delays + stealth |
| IP flag | Residential proxies (1 IP per account) |
| Fingerprint detection | playwright-stealth + realistic viewport |
| Session expiry | Persistent context + re-auth detection |
| LinkedIn ToS | Use throwaway accounts for scraping, never your main |
| Rate limit checkpoint | Detect + pause + wait 24h before resuming |

---

## Build Order (Recommended)

1. `scraper/session.py` — Account session manager (login once, persist to disk)
2. `scraper/browser.py` — Playwright launcher with stealth + anti-detection
3. `scraper/search.py` — People search (navigate + parse + paginate)
4. `scraper/profile.py` — Individual profile scraper (enrichment)
5. `scraper/pool.py` — Account pool with daily limit tracking
6. `scraper/pipeline.py` — Dedupe + store to database
7. `scraper/scheduler.py` — Distribute actions across the day

---

## Reference Projects

- `The-Web-Scraping-Playbook/awesome-linkedin-scrapers` — curated list of maintained scrapers
- `ManiMozaffar/linkedIn-scraper` — Playwright + FastAPI based
- `garaekz/inscraper` — TypeScript, Playwright, profile + experience pages
- `drissbri/linkedin-scraper` — Selenium + FastAPI with REST endpoints
