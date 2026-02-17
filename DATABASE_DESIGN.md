# Visa Document Validator — Database Design

**Status:** FINAL  
**Created:** 2026-02-16  
**Storage:** JSON files (no SQL database)

---

## 1. DATA MODEL OVERVIEW

```
┌─────────────────┐       ┌─────────────────┐
│    COUNTRY      │       │   VISA_TYPE     │
│─────────────────│       │─────────────────│
│ code (PK)       │       │ code (PK)       │
│ name            │       │ name            │
│ vwp_eligible    │       │ category        │
│ six_month_club  │       │ base_fee        │
│ restrictions[]  │       │ sevis_fee       │
└────────┬────────┘       │ petition_req    │
         │                │ dependent_visa  │
         │                └────────┬────────┘
         │                         │
         └────────────┬────────────┘
                      │
                      ▼
         ┌────────────────────────┐
         │      RECIPROCITY       │
         │────────────────────────│
         │ country_code (FK)      │
         │ visa_type (FK)         │
         │ fee                    │
         │ validity_months        │
         │ entries                │
         └────────────────────────┘
                      
         ┌────────────────────────┐
         │     REQUIREMENTS       │
         │────────────────────────│
         │ visa_type (FK)         │
         │ documents[]            │
         │ common_mistakes[]      │
         │ interview_tips[]       │
         └────────────────────────┘
```

---

## 2. ENTITY DEFINITIONS

### 2.1 Country
Represents a nationality/country of origin.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `code` | string(2) | ✅ | ISO 3166-1 alpha-2 (PK) |
| `name` | string | ✅ | Full country name |
| `vwp_eligible` | boolean | ✅ | Visa Waiver Program eligible |
| `six_month_club` | boolean | ✅ | Passport validity exemption |
| `vwp_restrictions` | string[] | ❌ | Countries that disqualify from VWP |
| `dual_nationality_block` | string[] | ❌ | Blocked dual nationalities |

### 2.2 Visa Type
Represents a US nonimmigrant visa category.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `code` | string | ✅ | Visa code (PK) e.g., "B1B2", "F1" |
| `name` | string | ✅ | Full name |
| `category` | enum | ✅ | tourist, student, work, exchange, family, other |
| `purpose` | string | ✅ | Purpose description |
| `base_fee` | integer | ✅ | Base MRV fee in USD |
| `sevis_fee` | integer | ❌ | SEVIS fee if applicable |
| `petition_required` | boolean | ✅ | Requires USCIS petition |
| `petition_form` | string | ❌ | Form number (e.g., "I-129") |
| `dol_required` | boolean | ✅ | Requires DOL certification |
| `required_forms` | string[] | ❌ | e.g., ["I-20"], ["DS-2019"] |
| `dependent_visa` | string | ❌ | Associated dependent visa code |
| `entry_restriction` | string | ❌ | e.g., "30 days before program" |
| `two_year_rule` | boolean | ✅ | 212(e) applies |
| `source` | string | ✅ | Official URL |

### 2.3 Reciprocity
Represents fee/validity for a country + visa type combination.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `country_code` | string(2) | ✅ | FK to Country |
| `visa_type` | string | ✅ | FK to Visa Type |
| `fee` | integer | ✅ | Reciprocity fee in USD |
| `validity_months` | integer | ✅ | Visa validity in months |
| `entries` | string | ✅ | "1", "2", or "M" (multiple) |
| `notes` | string | ❌ | Special conditions |

### 2.4 Document
Represents a required or supporting document.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | ✅ | Unique identifier |
| `name` | string | ✅ | Document name |
| `description` | string | ✅ | What it is |
| `required` | boolean | ✅ | Required vs supporting |
| `link` | string | ❌ | URL to form/info |
| `exception` | string | ❌ | When not needed |
| `must_sign` | boolean | ❌ | Requires signature |

### 2.5 Requirements
Represents complete requirements for a visa type.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `visa_type` | string | ✅ | FK to Visa Type (PK) |
| `documents.required` | Document[] | ✅ | Required documents |
| `documents.supporting` | Document[] | ✅ | Supporting documents |
| `common_mistakes` | string[] | ✅ | Common applicant errors |
| `interview_tips` | string[] | ❌ | Interview preparation |
| `source` | string | ✅ | Official URL |
| `last_updated` | date | ✅ | Data freshness |

### 2.6 Photo Specifications
Global photo requirements.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `size_inches` | string | ✅ | "2x2" |
| `size_mm` | string | ✅ | "51x51" |
| `background` | string | ✅ | "white or off-white" |
| `recency_months` | integer | ✅ | 6 |
| `format` | string | ✅ | "JPEG" |
| `dimensions_px` | string | ✅ | "600x600 to 1200x1200" |
| `max_file_size_kb` | integer | ✅ | 240 |
| `source` | string | ✅ | Official URL |

---

## 3. RELATIONSHIPS

```
Country (1) ──────────── (M) Reciprocity
                              │
Visa_Type (1) ────────── (M) ─┘

Visa_Type (1) ──────────── (1) Requirements
```

**Cardinality:**
- One Country has Many Reciprocity records (one per visa type)
- One Visa_Type has Many Reciprocity records (one per country)
- One Visa_Type has One Requirements record

**Composite Key:**
- Reciprocity: `(country_code, visa_type)`

---

## 4. JSON FILE MAPPING

| Entity | File(s) | Structure |
|--------|---------|-----------|
| Country | `meta/countries.json` | Array of countries |
| Country (VWP) | `meta/visa_waiver.json` | Object with VWP data |
| Country (6-month) | `meta/six_month_club.json` | Array of country codes |
| Visa_Type | `meta/visa_types.json` | Object keyed by visa code |
| Reciprocity | `reciprocity/{CC}.json` | One file per country |
| Requirements | `requirements/{VISA}.json` | One file per visa type |
| Photo Specs | `meta/photo_specs.json` | Single object |

---

## 5. DETAILED SCHEMAS

### 5.1 `meta/countries.json`
```json
{
  "lastUpdated": "2026-02-16",
  "source": "ISO 3166-1",
  "count": 200,
  "countries": [
    {
      "code": "AF",
      "name": "Afghanistan"
    },
    {
      "code": "AL", 
      "name": "Albania"
    }
  ]
}
```

### 5.2 `meta/visa_waiver.json`
```json
{
  "lastUpdated": "2026-02-16",
  "source": "https://travel.state.gov/.../visa-waiver-program.html",
  "eligibleCountries": [
    "AD", "AU", "AT", "BE", "BN", "CL", "HR", "CZ", "DK", "EE",
    "FI", "FR", "DE", "GR", "HU", "IS", "IE", "IT", "JP", "LV",
    "LI", "LT", "LU", "MT", "MC", "NL", "NZ", "NO", "PL", "PT",
    "QA", "SM", "SG", "SK", "SI", "KR", "ES", "SE", "CH", "TW", "GB"
  ],
  "restrictions": {
    "travelHistory": [
      "Traveled to DPRK, Iran, Iraq, Libya, Somalia, Sudan, Syria, or Yemen after March 1, 2011",
      "Traveled to Cuba after January 12, 2021"
    ],
    "dualNationality": [
      "Also a national of Cuba, DPRK, Iran, Iraq, Sudan, or Syria"
    ]
  }
}
```

### 5.3 `meta/six_month_club.json`
```json
{
  "lastUpdated": "2026-02-16",
  "source": "https://www.cbp.gov/.../six-month-club-update",
  "description": "Countries exempt from 6-month passport validity rule",
  "countries": [
    "AR", "AT", "AU", "BE", "BG", "BR", "CA", "CH", "CL", "CO",
    "CR", "CY", "CZ", "DE", "DK", "EE", "ES", "FI", "FR", "GB",
    "GR", "HR", "HU", "IE", "IL", "IT", "JP", "KR", "LT", "LU",
    "LV", "MT", "MX", "NL", "NO", "NZ", "PL", "PT", "RO", "SE",
    "SG", "SI", "SK", "TW"
  ]
}
```

### 5.4 `meta/visa_types.json`
```json
{
  "lastUpdated": "2026-02-16",
  "source": "travel.state.gov",
  "types": {
    "B1B2": {
      "code": "B1B2",
      "name": "Visitor Visa (B-1/B-2)",
      "category": "tourist_business",
      "purpose": "Tourism, Business, Medical Treatment",
      "baseFee": 185,
      "sevisFee": null,
      "petitionRequired": false,
      "petitionForm": null,
      "dolRequired": false,
      "requiredForms": [],
      "dependentVisa": null,
      "entryRestriction": null,
      "twoYearRule": false,
      "source": "https://travel.state.gov/content/travel/en/us-visas/tourism-visit/visitor.html"
    },
    "F1": {
      "code": "F1",
      "name": "Student Visa (F-1)",
      "category": "student",
      "purpose": "Academic Studies",
      "baseFee": 185,
      "sevisFee": 350,
      "petitionRequired": false,
      "petitionForm": null,
      "dolRequired": false,
      "requiredForms": ["I-20"],
      "dependentVisa": "F2",
      "entryRestriction": "Cannot enter more than 30 days before program start",
      "twoYearRule": false,
      "source": "https://travel.state.gov/content/travel/en/us-visas/study/student-visa.html"
    },
    "J1": {
      "code": "J1",
      "name": "Exchange Visitor (J-1)",
      "category": "exchange",
      "purpose": "Exchange Programs",
      "baseFee": 185,
      "sevisFee": 220,
      "petitionRequired": false,
      "petitionForm": null,
      "dolRequired": false,
      "requiredForms": ["DS-2019"],
      "dependentVisa": "J2",
      "entryRestriction": null,
      "twoYearRule": true,
      "source": "https://travel.state.gov/content/travel/en/us-visas/study/exchange.html"
    },
    "H1B": {
      "code": "H1B",
      "name": "Specialty Occupation (H-1B)",
      "category": "work",
      "purpose": "Specialty Occupation Employment",
      "baseFee": 205,
      "sevisFee": null,
      "petitionRequired": true,
      "petitionForm": "I-129",
      "dolRequired": true,
      "requiredForms": ["I-797"],
      "dependentVisa": "H4",
      "entryRestriction": null,
      "twoYearRule": false,
      "source": "https://travel.state.gov/content/travel/en/us-visas/employment/temporary-worker-visas.html"
    }
  }
}
```

### 5.5 `reciprocity/IN.json`
```json
{
  "countryCode": "IN",
  "countryName": "India",
  "lastUpdated": "2026-02-16",
  "source": "https://travel.state.gov/content/travel/en/us-visas/Visa-Reciprocity-and-Civil-Documents-by-Country/India.html",
  "vwpEligible": false,
  "sixMonthClub": false,
  "visaTypes": {
    "B1B2": {
      "fee": 0,
      "validityMonths": 120,
      "entries": "M",
      "notes": null
    },
    "F1": {
      "fee": 0,
      "validityMonths": 60,
      "entries": "M",
      "notes": null
    },
    "F2": {
      "fee": 0,
      "validityMonths": 60,
      "entries": "M",
      "notes": null
    },
    "J1": {
      "fee": 0,
      "validityMonths": 60,
      "entries": "M",
      "notes": null
    },
    "H1B": {
      "fee": 0,
      "validityMonths": 60,
      "entries": "M",
      "notes": null
    },
    "L1": {
      "fee": 0,
      "validityMonths": 60,
      "entries": "M",
      "notes": null
    }
  }
}
```

### 5.6 `requirements/B1B2.json`
```json
{
  "visaType": "B1B2",
  "lastUpdated": "2026-02-16",
  "source": "https://travel.state.gov/content/travel/en/us-visas/tourism-visit/visitor.html",
  "documents": {
    "required": [
      {
        "id": "passport",
        "name": "Valid Passport",
        "description": "Must be valid for at least 6 months beyond your intended period of stay in the United States",
        "exception": "Six Month Club countries are exempt from this requirement",
        "link": null
      },
      {
        "id": "ds160",
        "name": "DS-160 Confirmation Page",
        "description": "Complete the online nonimmigrant visa application and print the barcode confirmation page",
        "link": "https://ceac.state.gov/genniv/"
      },
      {
        "id": "photo",
        "name": "Passport-Style Photo",
        "description": "One 2x2 inch photo meeting US visa photo requirements",
        "link": "https://travel.state.gov/content/travel/en/us-visas/visa-information-resources/photos.html"
      },
      {
        "id": "fee_receipt",
        "name": "MRV Fee Payment Receipt",
        "description": "Proof of visa application fee payment ($185)",
        "link": null
      },
      {
        "id": "appointment",
        "name": "Interview Appointment Confirmation",
        "description": "Confirmation of scheduled interview at US Embassy or Consulate",
        "link": null
      }
    ],
    "supporting": [
      {
        "id": "employment",
        "name": "Employment Verification",
        "description": "Letter from employer stating position, salary, tenure, and approved leave dates",
        "purpose": "Demonstrates ties to home country and intent to return"
      },
      {
        "id": "bank_statements",
        "name": "Bank Statements",
        "description": "3-6 months of bank statements showing sufficient funds to cover trip expenses",
        "purpose": "Proves financial ability to fund the trip"
      },
      {
        "id": "itinerary",
        "name": "Travel Itinerary",
        "description": "Flight reservations, hotel bookings, and travel plans",
        "purpose": "Shows clear travel purpose and return plans",
        "cta": {
          "text": "Get verified flight reservation",
          "action": "dummyvisaticket"
        }
      },
      {
        "id": "property",
        "name": "Property Documents",
        "description": "Proof of property ownership, lease agreements, or vehicle registration",
        "purpose": "Demonstrates ties to home country"
      },
      {
        "id": "family_ties",
        "name": "Family Documentation",
        "description": "Marriage certificate, birth certificates of children, family photos",
        "purpose": "Demonstrates family ties to home country"
      },
      {
        "id": "invitation",
        "name": "Invitation Letter",
        "description": "Letter from US host (family, friend, or business) if applicable",
        "purpose": "Supports purpose of visit"
      },
      {
        "id": "previous_visas",
        "name": "Previous US Visas",
        "description": "Copies of previous US visas and passports if available",
        "purpose": "Shows travel history and compliance"
      }
    ]
  },
  "commonMistakes": [
    "Passport expiring within 6 months of travel date",
    "Forgetting to print DS-160 barcode confirmation page",
    "Photo does not meet specifications (wrong size, background, or age)",
    "Insufficient financial documentation",
    "No evidence of ties to home country",
    "Incomplete or inconsistent information on DS-160"
  ],
  "interviewTips": [
    "Bring all original documents plus copies",
    "Be prepared to explain purpose of trip clearly",
    "Have specific dates and itinerary ready",
    "Be honest and consistent with DS-160 answers",
    "Dress professionally"
  ]
}
```

### 5.7 `meta/photo_specs.json`
```json
{
  "lastUpdated": "2026-02-16",
  "source": "https://travel.state.gov/content/travel/en/us-visas/visa-information-resources/photos.html",
  "print": {
    "sizeInches": "2x2",
    "sizeMm": "51x51",
    "headHeightInches": "1 to 1-3/8",
    "eyeHeightInches": "1-1/8 to 1-3/8 from bottom"
  },
  "digital": {
    "format": "JPEG",
    "dimensionsPx": {
      "min": "600x600",
      "max": "1200x1200"
    },
    "maxFileSizeKb": 240,
    "colorDepth": "24-bit"
  },
  "requirements": {
    "background": "White or off-white",
    "recencyMonths": 6,
    "expression": "Neutral with both eyes open",
    "glasses": "Not allowed (removed October 2016)",
    "headCoverings": "Only for religious reasons",
    "lighting": "No shadows on face or background"
  }
}
```

---

## 6. INDEXES (Logical)

For fast lookups in JavaScript:

```javascript
// Primary indexes (built at load time)
const countryIndex = {};      // code -> country object
const visaTypeIndex = {};     // code -> visa type object
const reciprocityIndex = {};  // countryCode -> visa data

// Example usage
countryIndex["IN"]           // { code: "IN", name: "India" }
visaTypeIndex["F1"]          // { code: "F1", name: "Student Visa", ... }
reciprocityIndex["IN"]["F1"] // { fee: 0, validityMonths: 60, entries: "M" }
```

---

## 7. DATA INTEGRITY RULES

| Rule | Description |
|------|-------------|
| **Country code format** | Must be 2-letter ISO 3166-1 alpha-2 |
| **Visa code format** | Must match official code (B1B2, F1, H1B, etc.) |
| **Fee values** | Must be non-negative integers |
| **Validity** | Must be positive integer (months) |
| **Entries** | Must be "1", "2", or "M" |
| **Source URLs** | Must be valid travel.state.gov URLs |
| **Last updated** | Must be ISO date format (YYYY-MM-DD) |

---

## 8. DATA VOLUME

| Entity | Count | Storage |
|--------|-------|---------|
| Countries | ~200 | ~10 KB |
| Visa Types | ~40 | ~15 KB |
| Reciprocity | ~200 files × ~40 types | ~500 KB |
| Requirements | ~40 files | ~200 KB |
| Meta files | 4 | ~20 KB |
| **Total** | | **~750 KB** |

---

## 9. LINK STRATEGY

**Decision:** Minimal links to reduce maintenance burden.

| Link Type | Include? | Rationale |
|-----------|----------|-----------|
| DS-160 (ceac.state.gov/genniv/) | ✅ Yes | Stable 10+ years |
| SEVIS payment (fmjfee.com) | ✅ Yes | Official, stable |
| Photo specs page | ✅ Yes | Rarely changes |
| Form PDFs | ❌ No | URLs change often |
| Visa info pages | ❌ No | Site restructures |

**For forms:** Provide name/number only (e.g., "Form I-20") — user can search.

---

## 10. UPDATE STRATEGY

| Data | Update Frequency | Method |
|------|------------------|--------|
| Countries | Never | Static |
| VWP list | Annually | Manual review |
| Six Month Club | Annually | Manual review |
| Visa types | Rarely | Manual review |
| Reciprocity | Annually | Automated scrape + review |
| Requirements | Rarely | Manual review |
| Photo specs | Rarely | Manual review |

**Automated Pipeline:**
```
Weekly Cron
    │
    ▼
Scrape travel.state.gov
    │
    ▼
Diff against existing data
    │
    ▼
Changes detected?
    │
    ├── No → Done
    │
    └── Yes → Alert for manual review
              │
              ▼
         Manual approval
              │
              ▼
         Merge to data/
```

---

**Database Design Complete ✅**

*Next: Implementation*

---

*Database Design by Cookie 🍪 | 2026-02-16*
