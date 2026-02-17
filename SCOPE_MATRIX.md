# Visa Validator — Complete Scope Matrix

**Principle:** MVP = Complete. Every visa type × Every country × Every requirement.

---

## 1. ALL US VISA TYPES (Nonimmigrant)

### Tourist/Business
| Code | Name | Base Fee | Petition? | Notes |
|------|------|----------|-----------|-------|
| B-1 | Business Visitor | $185 | No | |
| B-2 | Tourist/Medical | $185 | No | |
| B-1/B-2 | Combined | $185 | No | Most common |

### Student/Exchange
| Code | Name | Base Fee | Petition? | Notes |
|------|------|----------|-----------|-------|
| F-1 | Academic Student | $185 | No | Requires I-20, SEVIS ($350) |
| F-2 | F-1 Dependent | $185 | No | Spouse/child of F-1 |
| M-1 | Vocational Student | $185 | No | Requires I-20, SEVIS ($350) |
| M-2 | M-1 Dependent | $185 | No | |
| J-1 | Exchange Visitor | $185 | No | Requires DS-2019, SEVIS ($220), 212(e) rule |
| J-2 | J-1 Dependent | $185 | No | |

### Work Visas (Petition-Based)
| Code | Name | Base Fee | Petition? | Notes |
|------|------|----------|-----------|-------|
| H-1B | Specialty Occupation | $205 | Yes (USCIS) | DOL certification required |
| H-1B1 | Chile/Singapore FTA | $205 | Yes | DOL required |
| H-2A | Temporary Agricultural | $205 | Yes | DOL required |
| H-2B | Temporary Non-Ag | $205 | Yes | DOL required |
| H-3 | Trainee | $205 | Yes | |
| H-4 | H Dependent | $205 | No | |
| L-1A | Intracompany Manager | $205 | Yes | |
| L-1B | Intracompany Specialized | $205 | Yes | |
| L-2 | L Dependent | $205 | No | |
| O-1A | Extraordinary Ability (Science/Business) | $205 | Yes | |
| O-1B | Extraordinary Ability (Arts) | $205 | Yes | |
| O-2 | O-1 Support | $205 | Yes | |
| O-3 | O Dependent | $205 | No | |
| P-1A | Athlete | $205 | Yes | |
| P-1B | Entertainment Group | $205 | Yes | |
| P-2 | Reciprocal Exchange Artist | $205 | Yes | |
| P-3 | Culturally Unique Artist | $205 | Yes | |
| P-4 | P Dependent | $205 | No | |
| Q-1 | Cultural Exchange | $205 | Yes | |
| R-1 | Religious Worker | $205 | Yes | |
| R-2 | R Dependent | $205 | No | |

### Treaty Visas
| Code | Name | Base Fee | Petition? | Notes |
|------|------|----------|-----------|-------|
| E-1 | Treaty Trader | $315 | No | Nationality-restricted |
| E-2 | Treaty Investor | $315 | No | Nationality-restricted |
| E-3 | Australian Specialty | $315 | No | Australia only, DOL |

### Family/Fiancé
| Code | Name | Base Fee | Petition? | Notes |
|------|------|----------|-----------|-------|
| K-1 | Fiancé(e) | $265 | Yes (USCIS) | |
| K-2 | K-1 Child | $265 | No | |
| K-3 | Spouse of USC | $265 | Yes | |
| K-4 | K-3 Child | $265 | No | |

### Other
| Code | Name | Base Fee | Petition? | Notes |
|------|------|----------|-----------|-------|
| C-1 | Transit | $185 | No | |
| C-1/D | Transit/Crewmember | $185 | No | |
| D | Crewmember | $185 | No | |
| I | Media/Journalist | $185 | No | |
| TN | NAFTA Professional | $185 | No | Canada/Mexico only |
| TD | TN Dependent | $185 | No | |

### Diplomatic/Official (No Fee)
| Code | Name | Base Fee | Notes |
|------|------|----------|-------|
| A-1 | Ambassador/Diplomat | $0 | |
| A-2 | Other Official | $0 | |
| A-3 | A Attendant | $0 | |
| G-1 to G-5 | Intl Org Rep | $0 | |
| NATO | NATO Personnel | $0 | |

### Victim/Witness
| Code | Name | Base Fee | Notes |
|------|------|----------|-------|
| T | Trafficking Victim | $185 | USCIS petition |
| U | Crime Victim | $185 | USCIS petition |
| S | Witness/Informant | $185 | |

---

## 2. ALL COUNTRIES (~200)

Full ISO 3166-1 list. Each country has:
- Country code (2-letter)
- Country name
- Visa Waiver Program status
- Six Month Club status
- Reciprocity data per visa type:
  - Fee
  - Validity (months)
  - Entries (1, 2, M)
- Special restrictions (VWP disqualifiers)

---

## 3. COMPLETE REQUIREMENTS PER VISA TYPE

### Universal Documents (All Visas)
| Document | Description |
|----------|-------------|
| Valid Passport | 6+ months validity (unless Six Month Club exempt) |
| DS-160 Confirmation | Online form with barcode page |
| Passport Photo | 2x2", white background, 6 months recent |
| Fee Payment Receipt | MRV fee proof |
| Appointment Confirmation | Interview scheduled |

### Visa-Specific Documents

**B-1/B-2:**
- Employment letter
- Bank statements (3-6 months)
- Travel itinerary
- Property/ties documentation
- Invitation letter (if applicable)

**F-1:**
- Form I-20 (from school)
- SEVIS fee receipt ($350)
- Financial evidence (tuition + living expenses)
- Transcripts
- Test scores (if applicable)
- School acceptance letter

**J-1:**
- Form DS-2019 (from sponsor)
- SEVIS fee receipt ($220)
- Sponsor letter
- Program details
- Financial evidence
- 212(e) applicability check

**H-1B:**
- Approved I-129 petition
- I-797 approval notice
- LCA (Labor Condition Application)
- Employer letter
- Degree/credentials
- Resume/CV

**L-1:**
- Approved I-129 petition
- I-797 approval notice
- Evidence of qualifying relationship
- Employment history
- Position details

*(Continue for each visa type...)*

---

## 4. DATA MATRIX STRUCTURE

```
MATRIX = {
  "US": {
    "visa_types": {
      "B1B2": {
        "requirements": [...],
        "fees": { "base": 185, "sevis": 0 },
        "countries": {
          "IN": { "reciprocity_fee": 0, "validity": 120, "entries": "M" },
          "CN": { "reciprocity_fee": 185, "validity": 120, "entries": "M" },
          ...
        }
      },
      "F1": {
        "requirements": [...],
        "fees": { "base": 185, "sevis": 350 },
        "countries": { ... }
      },
      ...
    }
  }
}
```

**Total Combinations:**
- ~40 visa types × ~200 countries = **~8,000 combinations**
- Each combination needs: fee, validity, entries, restrictions

---

## 5. ADDITIONAL DATA POINTS (per Sub-Agent Analysis)

### Per Country
| Field | Description |
|-------|-------------|
| `visaWaiverEligible` | VWP status |
| `sixMonthClub` | Passport validity exemption |
| `vwpRestrictions` | Countries that disqualify from VWP |
| `dualNationalityRestrictions` | Blocked dual nationalities |
| `interviewWaiverEligible` | Renewal without interview |

### Per Visa Type
| Field | Description |
|-------|-------------|
| `sevisFee` | F/M/J visa SEVIS fee |
| `petitionRequired` | Needs USCIS petition first |
| `dolRequired` | Needs DOL certification |
| `maxStayDuration` | Maximum time in US |
| `dependentVisa` | Associated dependent visa code |
| `twoYearRule` | 212(e) applies (J-1) |
| `entryTimingRestriction` | e.g., F-1 30-day rule |

### Photo Requirements
| Field | Value |
|-------|-------|
| `size` | 2x2 inches (51x51mm) |
| `background` | White or off-white |
| `recency` | Within 6 months |
| `format` | JPEG, 600x600 to 1200x1200 pixels |
| `fileSize` | ≤240KB |

---

## 6. REVISED DELIVERABLES

### Data Files
```
data/
├── countries.json                    # All 200 countries
├── visa_waiver.json                  # VWP countries + restrictions
├── six_month_club.json               # Passport validity exemptions
├── visa_types.json                   # All ~40 visa types
├── photo_requirements.json           # Photo specs
│
├── requirements/                     # Per visa type
│   ├── B1B2.json
│   ├── F1.json
│   ├── J1.json
│   ├── H1B.json
│   ├── L1.json
│   ├── ... (all visa types)
│
└── reciprocity/                      # Per country
    ├── AF.json                       # All visa types for Afghanistan
    ├── AL.json
    ├── ... (~200 files)
```

### Module Functions
```javascript
// Complete API
getVisaRequirements(nationality, visaType, destination)
isVisaWaiverEligible(nationality, travelHistory)
getDocumentChecklist(visaType)
getFees(nationality, visaType)  // base + reciprocity + SEVIS
getValidity(nationality, visaType)
getPhotoRequirements()
getDependentVisa(visaType)
checkTwoYearRule(visaType, nationality)
getSixMonthClubStatus(nationality)
```

---

## 7. TIMELINE IMPACT

**Original estimate:** 1.5-2 weeks

**Revised estimate:** 3-4 weeks
- Week 1: Scrape ALL reciprocity data (200 countries × 40 visa types)
- Week 2: Write ALL requirement checklists (40 visa types)
- Week 3: Build module + validation
- Week 4: Testing + documentation

---

## 8. NEXT STEPS

1. ✅ Martin confirms complete scope
2. Start with data infrastructure:
   - Build robust scraper for all visa types
   - Create complete country list
   - Scrape all reciprocity tables
3. Document all visa requirements (40 types)
4. Build validation module

---

*Scope Matrix by Cookie 🍪 | 2026-02-16*
