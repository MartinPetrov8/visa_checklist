# Visa Document Validator — Discovery

## User Persona

**Primary User:** Visa applicant preparing documents for US visa application

**Characteristics:**
- First-time or infrequent visa applicant
- Confused by scattered/unclear embassy requirements
- Anxious about rejection due to missing documents
- Willing to pay for peace of mind

**Where they are today:**
- Googling "US visa requirements"
- Reading embassy websites (confusing, buried info)
- Asking in forums/Reddit
- Paying immigration consultants ($200+)

---

## Problem Statement

Visa applicants don't know if their document package is complete before submitting. Requirements are scattered across multiple pages, vary by visa type, and embassy websites are hard to navigate. 

**Result:** Rejections, wasted fees, rebooking costs, stress.

---

## User Stories

### MVP User Stories

**US-1: Check if I need a visa**
> As a traveler planning a US trip,  
> I want to know if I need a visa or can use ESTA,  
> So that I don't waste time preparing unnecessary documents.

**US-2: Get document checklist**
> As a visa applicant,  
> I want to see all required documents for my visa type,  
> So that I can prepare a complete application.

**US-3: Understand what each document means**
> As a first-time applicant,  
> I want explanations for each required document,  
> So that I know exactly what to prepare.

**US-4: Know the fees**
> As an applicant,  
> I want to know total fees (base + reciprocity) for my nationality,  
> So that I can budget correctly.

**US-5: Know visa validity**
> As an applicant,  
> I want to know how long my visa will be valid and how many entries I get,  
> So that I can plan future trips.

**US-6: See the sources**
> As an applicant,  
> I want to see official sources for all information,  
> So that I can verify and trust the data.

**US-7: Get a printable checklist**
> As an applicant,  
> I want to download/print my checklist with sources,  
> So that I can track what I've gathered and verify info.

### Future User Stories (Phase 2+)

**US-8: Validate my documents**
> As an applicant with documents ready,  
> I want to check if my documents meet requirements,  
> So that I catch problems before submitting.

---

## MVP Scope

### In Scope
| Feature | Description |
|---------|-------------|
| Nationality selection | Dropdown with all countries |
| Visa type selection | Multiple types (B1/B2, F1, H1B, etc.) |
| Visa waiver check | Tell user if they don't need visa |
| Document checklist | List all required + supporting docs |
| Document explanations | What each document is and why needed |
| **Fees breakdown** | Base fee + reciprocity fee by nationality |
| **Visa validity** | Duration + number of entries by nationality |
| **Source citations** | Show official source for all info |
| **Downloadable output** | PDF/printable with sources included |

### Out of Scope (Phase 1)
| Feature | Why Out |
|---------|---------|
| Wait times | Volatile, maintenance burden |
| Residence country | Doesn't affect requirements |
| Document upload | Complex, Phase 2+ |
| UK/Schengen | Start with US only |

---

## Visa Types to Support

### Phase 1 (MVP)
| Visa | Name | Use Case | Base Fee |
|------|------|----------|----------|
| **B1/B2** | Visitor | Tourism, business meetings | $185 |
| **F1** | Student | Academic studies | $185 |
| **J1** | Exchange Visitor | Work/study exchange programs | $185 |

### Phase 2
| Visa | Name | Use Case | Base Fee |
|------|------|----------|----------|
| **H1B** | Specialty Occupation | Skilled workers | $205 |
| **L1** | Intra-company Transfer | Company transfers | $205 |
| **O1** | Extraordinary Ability | Artists, scientists, athletes | $205 |
| **K1** | Fiancé(e) | Marriage to US citizen | $265 |

*Each visa type has different document requirements — need separate checklist per type.*

---

## Acceptance Criteria

### US-1: Check if I need a visa
- [ ] User can select nationality from complete list
- [ ] System correctly identifies Visa Waiver countries
- [ ] Visa Waiver users see "ESTA" guidance, not visa docs

### US-2: Get document checklist
- [ ] Shows all required documents for B1/B2
- [ ] Shows supporting documents (recommended)
- [ ] Distinguishes required vs recommended

### US-3: Understand documents
- [ ] Each document has clear description
- [ ] Common mistakes highlighted
- [ ] Links to official forms (DS-160, etc.)

---

## Open Questions

| Question | Status | Answer |
|----------|--------|--------|
| Do docs differ by nationality? | ✅ Answered | No — same for all |
| Does residence country matter? | ✅ Answered | No — ignore for MVP |
| How often do requirements change? | ✅ Answered | Rarely (annually at most) |
| What visa types to support? | ✅ Answered | Multiple: B1/B2, F1, H1B, J1, etc. |
| Monetization model? | ✅ Answered | Add-on to dummyvisaticket |

---

## Monetization

**Model:** Bundled add-on to dummyvisaticket.com

**Flow:**
1. User validates documents (free or low-cost entry)
2. Cross-sell: "Need a flight reservation for your visa? → dummyvisaticket"
3. Bundle pricing: Validator + Dummy Ticket = discount

---

## Data Requirements

| Data | Source | Volume | Update Frequency |
|------|--------|--------|------------------|
| Country list | ISO 3166 | ~200 | Never |
| Visa Waiver countries | travel.state.gov | ~41 | Rarely |
| Visa types | travel.state.gov | ~20 categories | Rarely |
| Document checklist per visa | travel.state.gov | 1 per visa type | Rarely |
| **Fees by nationality** | travel.state.gov/reciprocity | ~200 countries × visa types | Annually |
| **Validity by nationality** | travel.state.gov/reciprocity | ~200 countries × visa types | Annually |

### Source URLs (must be cited in output)
- Base fees: `travel.state.gov/us-visas/visa-information-resources/fees/fees-visa-services.html`
- Reciprocity: `travel.state.gov/us-visas/Visa-Reciprocity-and-Civil-Documents-by-Country/{COUNTRY}.html`
- Requirements: `travel.state.gov/us-visas/tourism-visit/visitor.html` (and per visa type)

---

## Success Metrics

| Metric | Target |
|--------|--------|
| Users complete checklist | 100 in first month |
| Accuracy complaints | <5% |
| Time to complete | <2 minutes |

---

## Next Steps

1. ✅ **Scope confirmed** — B1/B2, F1, J1 for MVP; add more in Phase 2
2. ✅ **Monetization confirmed** — Add-on to dummyvisaticket
3. **Build data:**
   - [ ] Compile country list (ISO 3166)
   - [ ] Compile Visa Waiver country list
   - [ ] Scrape reciprocity data (fees + validity) for all countries
   - [ ] Write document checklists for B1/B2, F1, J1
4. **Build UI:**
   - [ ] Nationality dropdown
   - [ ] Visa type selector
   - [ ] Results page with checklist + fees + validity + sources
   - [ ] Download/print as PDF

---

---

## Edge Cases

| Case | How We Handle |
|------|---------------|
| **Dual nationals** | Show results for selected nationality; add note "If you hold multiple passports, check requirements for each" |
| **Stateless persons** | Show message: "Contact embassy directly for stateless travel documents" |
| **Special territories** | Include separately (Hong Kong, Macau, Taiwan, Puerto Rico, etc.) |
| **Missing data** | Show disclaimer: "Data not available for this combination. Check official source: [link]" |
| **Data outdated** | Show last_updated date; add "Verify with embassy before applying" |

---

## Legal Disclaimer (Required)

> **Disclaimer:** This tool provides general information only and does not constitute legal advice. Visa requirements change frequently. Always verify requirements with the official embassy or consulate before submitting your application. We are not responsible for visa denials or application issues.

---

## Technical Requirements

| Requirement | Details |
|-------------|---------|
| **Hosting** | Static site (GitHub Pages or similar) |
| **Data updates** | Cron job weekly to check for changes |
| **Mobile** | Mobile-first responsive design |
| **Languages** | English only (MVP) |
| **PDF export** | Generate downloadable checklist with sources |

---

*Discovery by Cookie 🍪 | 2026-02-16*
