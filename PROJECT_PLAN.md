# Visa Document Validator — Project Plan
**Created:** 2026-02-16  
**Status:** Discovery Phase

---

## 1. DATA SOURCES ASSESSMENT

### Primary Source: travel.state.gov (US State Department)

| Page | URL | Reliability | Update Frequency | Scrapeable? |
|------|-----|-------------|------------------|-------------|
| **Visa Types** | `/us-visas/visa-information-resources/all-visa-categories.html` | ✅ Official | Rarely changes | Yes - HTML table |
| **Requirements** | `/us-visas/tourism-visit/visitor.html` | ✅ Official | Rarely changes | Yes - static content |
| **Fees** | `/us-visas/visa-information-resources/fees/fees-visa-services.html` | ✅ Official | Changes with policy | Yes - HTML list |
| **Reciprocity (per country)** | `/us-visas/Visa-Reciprocity-and-Civil-Documents-by-Country/{COUNTRY}.html` | ✅ Official | ~Annually | Yes - per country page |
| **Wait Times** | `/us-visas/visa-information-resources/global-visa-wait-times.html` | ✅ Official | **Monthly** (last: Feb 13, 2026) | Yes - HTML table |
| **Visa Waiver Countries** | `/us-visas/tourism-visit/visa-waiver-program.html` | ✅ Official | Rarely changes | Yes - list |

### Key Findings on Data Stability

1. **Core requirements DON'T change often**
   - DS-160 form, passport validity, photo specs — stable for years
   - Fee structure — changes maybe once per year with policy updates
   
2. **Reciprocity fees/validity — country-specific, stable**
   - Changes when bilateral agreements change (rare)
   - ~200 country pages to scrape once, then monitor
   
3. **Wait times — VOLATILE**
   - Updated monthly
   - Embassy-specific, not nationality-specific
   - "Estimate only, does not guarantee availability"
   - Can vary from <0.5 month to 16+ months (Abu Dhabi B1/B2)

### Secondary Sources (NOT recommended for MVP)

| Source | Issue |
|--------|-------|
| Embassy websites (usembassy.gov) | 200+ sites, inconsistent formats |
| USTravelDocs.com | Third-party, requires JS rendering |
| VisaJourney, ImmiHelp forums | User-generated, not authoritative |

---

## 2. OPEN QUESTIONS — ANSWERED

### Q1: Do requirements differ by nationality?
**Answer: NO** — Core document checklist is identical for all B1/B2 applicants.

**What DOES differ by nationality:**
- Whether visa is needed at all (Visa Waiver Program)
- Reciprocity fee (extra $0-$303 on top of $185)
- Visa validity period (12 months to 120 months)
- Number of entries (single vs multiple)

### Q2: Does residence country (where you apply from) affect requirements?
**Answer: NO** — Requirements stay the same.

**What DOES differ by location:**
- Wait times (embassy-specific)
- Difficulty proving "ties to home country" (practical, not legal)

**Official State Dept position:**
> "You should generally schedule an appointment at the U.S. Embassy or Consulate in the country where you live. You may schedule at another Embassy but it may be more difficult to demonstrate your qualifications."

**Recommendation for MVP:** DON'T include residence country. Too complex, minimal user value.

### Q3: Should we include wait times?
**Pros:**
- Valuable for planning
- Official data available (monthly updates)

**Cons:**
- Volatile — changes weekly in practice
- Per-embassy, not per-nationality
- Creates maintenance burden (monthly scrapes)
- "Estimate only" — legal disclaimer

**Recommendation:** 
- **Phase 1:** NO — too volatile, creates expectation we can't meet
- **Phase 2:** Maybe link to official page, don't promise our own data

---

## 3. SIMPLIFIED MVP SCOPE

### What We Build (Phase 1 — US Only)

```
INPUT:
├── Destination: US (fixed for MVP)
├── Nationality: [dropdown - ~200 countries]
├── Visa purpose: Tourist | Business | Both
└── Trip details: duration, dates (optional)

OUTPUT:
├── Tier: "Visa Waiver (ESTA)" | "Visa Required"
├── If Visa Required:
│   ├── Required Documents Checklist
│   ├── Fees: Base ($185) + Reciprocity (if any)
│   ├── Visa validity: X months, Y entries
│   └── Common mistakes for this visa type
└── Next steps with links to official resources
```

### What We DON'T Build (Phase 1)

| Feature | Why Not |
|---------|---------|
| Wait times | Too volatile, maintenance burden |
| Residence country selection | Doesn't affect requirements |
| Multiple visa types | Start simple (B1/B2 only) |
| UK destination | Phase 2 |
| Document upload/validation | Phase 2 |
| Embassy-specific info | 200+ embassies = scope creep |

---

## 4. DATA MODEL (Phase 1)

### File 1: `data/visa_waiver_countries.json`
```json
{
  "program": "US_VISA_WAIVER",
  "last_updated": "2026-02-16",
  "source": "travel.state.gov",
  "countries": ["AD", "AU", "AT", "BE", "BN", "CL", ...]
}
```
*~41 countries, rarely changes*

### File 2: `data/reciprocity/{country_code}.json`
```json
{
  "country_code": "IN",
  "country_name": "India",
  "last_updated": "2026-02-16",
  "source": "travel.state.gov/Visa-Reciprocity-and-Civil-Documents-by-Country/India.html",
  "visa_types": {
    "B1/B2": {
      "fee": 0,
      "validity_months": 120,
      "entries": "M"
    }
  }
}
```
*~200 files, one per country*

### File 3: `data/requirements/us_b1b2.json`
```json
{
  "visa_type": "B1/B2",
  "destination": "US",
  "last_updated": "2026-02-16",
  "base_fee_usd": 185,
  "documents": {
    "required": [
      {
        "name": "Valid Passport",
        "description": "Must be valid for at least 6 months beyond your period of stay",
        "exceptions": ["Six Month Club countries exempt from this rule"]
      },
      {
        "name": "DS-160 Confirmation",
        "description": "Online Nonimmigrant Visa Application form",
        "link": "https://ceac.state.gov/genniv/"
      },
      ...
    ],
    "supporting": [
      {
        "name": "Proof of Ties to Home Country",
        "description": "Employment letter, property ownership, family ties",
        "why": "Demonstrate intent to return after visit"
      },
      ...
    ]
  },
  "common_mistakes": [
    "Passport expires within 6 months of travel",
    "Missing DS-160 barcode page",
    "Insufficient financial documentation"
  ]
}
```

---

## 5. TECHNICAL APPROACH

### Scraper Strategy

| Data | Frequency | Method |
|------|-----------|--------|
| Visa Waiver list | Once, then quarterly check | Manual or simple scrape |
| Reciprocity tables | Once, then quarterly check | Scrape all ~200 country pages |
| Requirements | Once, manual entry | Static content, just write it |
| Fees | Once, then annual check | Simple scrape |

### Stack (Simple)

```
Frontend: Static HTML/JS (GitHub Pages compatible)
Backend: None (all logic client-side)
Data: JSON files, version-controlled
Updates: Manual or cron scraper → PR → review → merge
```

### Why No Backend?
- Zero hosting cost
- No API rate limits
- Data changes slowly (quarterly at most)
- Can always add backend later if needed

---

## 6. BUILD PHASES

### Phase 1: US B1/B2 Validator (2-3 weeks)
1. [ ] Scrape/compile Visa Waiver country list
2. [ ] Scrape reciprocity data for top 50 countries
3. [ ] Write US B1/B2 requirements checklist (manual)
4. [ ] Build simple web UI (select nationality → get checklist)
5. [ ] Deploy to GitHub Pages
6. [ ] Test with 10 real use cases

### Phase 2: Expand (2-3 weeks)
1. [ ] Complete reciprocity data for all ~200 countries
2. [ ] Add UK Standard Visitor visa
3. [ ] Add fee calculator (base + reciprocity)
4. [ ] Add "Download checklist as PDF" feature

### Phase 3: Integration (if dummyvisaticket ready)
1. [ ] Bundle as "Validate → Book" flow
2. [ ] Cross-sell integration
3. [ ] Shared customer database

---

## 7. RISKS & MITIGATIONS

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| State Dept changes website structure | Low | Version-controlled data, easy to re-scrape |
| Requirements change | Low | Data has `last_updated` field, show disclaimer |
| User relies on outdated data | Medium | Link to official source, add "verify before submitting" |
| Legal liability for wrong info | Medium | Clear disclaimer: "informational only, verify with embassy" |

---

## 8. SUCCESS METRICS

### MVP (Phase 1)
- [ ] 50 users try the validator
- [ ] <5 complaints about incorrect information
- [ ] Data covers top 50 nationalities (90%+ of visa applicants)

### Growth (Phase 2+)
- [ ] Convert 5% to dummyvisaticket customers
- [ ] 500 monthly active users

---

## 9. NEXT STEPS (Immediate)

1. **Build scraper** for reciprocity tables (top 20 countries first)
2. **Write requirements checklist** for US B1/B2 (manual, from official source)
3. **Create Visa Waiver country list** (quick, ~41 countries)
4. **Design simple UI** (mobile-first, single page)

---

## 10. DECISION POINTS FOR MARTIN

Before building, confirm:

1. **Scope:** US B1/B2 only for Phase 1? ✅ / ❌
2. **Wait times:** Exclude from Phase 1? ✅ / ❌  
3. **Residence country:** Ignore for MVP? ✅ / ❌
4. **Hosting:** GitHub Pages (free) vs paid hosting?
5. **Timeline:** 2-3 weeks acceptable for Phase 1?

---

*Plan created by Cookie 🍪 — Ready to proceed on confirmation*
