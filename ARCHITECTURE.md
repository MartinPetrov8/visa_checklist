# Visa Document Validator — Architecture Design

**Status:** FINAL  
**Approved:** 2026-02-16  
**Owner:** Cookie 🍪

---

## 1. CONTEXT

**What:** Plug-and-play JavaScript module for dummyvisaticket.com  
**Integration point:** Checkout flow — user selects nationality + visa type → returns complete checklist  
**Scope:** ALL US visa types × ALL countries = complete coverage

---

## 2. SYSTEM ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────────┐
│                    DUMMYVISATICKET.COM                          │
│                                                                  │
│   [Checkout Flow] ──→ [VISA VALIDATOR MODULE] ──→ [Results]    │
│                              │                                   │
└──────────────────────────────│───────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────┐
│                    VISA VALIDATOR MODULE                         │
│                                                                  │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│   │  visa-      │    │   DATA      │    │  OUTPUT     │        │
│   │  validator  │───▶│   LAYER     │───▶│  FORMATTER  │        │
│   │  .js        │    │  (JSON)     │    │             │        │
│   └─────────────┘    └─────────────┘    └─────────────┘        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. DATA ARCHITECTURE

### File Structure
```
data/
├── meta/
│   ├── countries.json           # ISO 3166 list (~200)
│   ├── visa_types.json          # All US visa types (~40)
│   ├── visa_waiver.json         # VWP countries + restrictions
│   ├── six_month_club.json      # Passport validity exemptions
│   └── photo_specs.json         # Photo requirements
│
├── requirements/                 # Document checklists per visa
│   ├── B1B2.json
│   ├── F1.json
│   ├── F2.json
│   ├── J1.json
│   ├── J2.json
│   ├── H1B.json
│   ├── H4.json
│   ├── L1A.json
│   ├── L1B.json
│   ├── L2.json
│   ├── O1.json
│   ├── K1.json
│   ├── E1.json
│   ├── E2.json
│   └── ... (all ~40 types)
│
└── reciprocity/                  # Per country fees/validity
    ├── AF.json
    ├── AL.json
    ├── AR.json
    └── ... (~200 files)
```

### Schema: `countries.json`
```json
{
  "lastUpdated": "2026-02-16",
  "source": "ISO 3166-1",
  "countries": [
    { "code": "AF", "name": "Afghanistan" },
    { "code": "AL", "name": "Albania" },
    ...
  ]
}
```

### Schema: `visa_types.json`
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
      "petitionRequired": false,
      "sevisFee": null,
      "dependentVisa": null,
      "source": "https://travel.state.gov/..."
    },
    "F1": {
      "code": "F1",
      "name": "Student Visa (F-1)",
      "category": "student",
      "purpose": "Academic Studies",
      "baseFee": 185,
      "petitionRequired": false,
      "sevisFee": 350,
      "requiredForms": ["I-20"],
      "dependentVisa": "F2",
      "entryRestriction": "30 days before program start",
      "source": "https://travel.state.gov/..."
    },
    "H1B": {
      "code": "H1B",
      "name": "Specialty Occupation (H-1B)",
      "category": "work",
      "purpose": "Specialty Occupation Work",
      "baseFee": 205,
      "petitionRequired": true,
      "petitionForm": "I-129",
      "dolRequired": true,
      "dependentVisa": "H4",
      "source": "https://travel.state.gov/..."
    }
  }
}
```

### Schema: `requirements/F1.json`
```json
{
  "visaType": "F1",
  "lastUpdated": "2026-02-16",
  "source": "https://travel.state.gov/content/travel/en/us-visas/study/student-visa.html",
  "documents": {
    "required": [
      {
        "id": "passport",
        "name": "Valid Passport",
        "description": "Valid for 6+ months beyond intended stay",
        "exception": "Six Month Club countries exempt"
      },
      {
        "id": "ds160",
        "name": "DS-160 Confirmation",
        "description": "Online application with barcode page",
        "link": "https://ceac.state.gov/genniv/"
      },
      {
        "id": "photo",
        "name": "Passport Photo",
        "description": "Per US visa photo requirements",
        "link": "https://travel.state.gov/.../photos.html"
      },
      {
        "id": "i20",
        "name": "Form I-20",
        "description": "Certificate of Eligibility from SEVP-certified school",
        "mustSign": true
      },
      {
        "id": "sevis_receipt",
        "name": "SEVIS Fee Receipt",
        "description": "I-901 SEVIS fee payment confirmation ($350)",
        "link": "https://www.fmjfee.com/"
      },
      {
        "id": "fee_receipt",
        "name": "Visa Fee Receipt",
        "description": "MRV fee payment proof"
      }
    ],
    "supporting": [
      {
        "id": "financial",
        "name": "Financial Evidence",
        "description": "Bank statements, sponsor letter, scholarship proof covering tuition + living expenses"
      },
      {
        "id": "transcripts",
        "name": "Academic Transcripts",
        "description": "Previous academic records"
      },
      {
        "id": "test_scores",
        "name": "Test Scores",
        "description": "TOEFL, GRE, GMAT if applicable"
      },
      {
        "id": "ties",
        "name": "Ties to Home Country",
        "description": "Evidence of intent to return after studies"
      }
    ]
  },
  "commonMistakes": [
    "Unsigned I-20",
    "SEVIS fee not paid before interview",
    "Insufficient financial documentation",
    "Arriving more than 30 days before program start"
  ],
  "interviewTips": [
    "Be prepared to explain your study plans",
    "Know your school and program details",
    "Demonstrate intent to return home after studies"
  ]
}
```

### Schema: `reciprocity/IN.json`
```json
{
  "countryCode": "IN",
  "countryName": "India",
  "lastUpdated": "2026-02-16",
  "source": "https://travel.state.gov/.../India.html",
  "visaWaiverEligible": false,
  "sixMonthClub": false,
  "visaTypes": {
    "B1B2": { "fee": 0, "validity": 120, "entries": "M" },
    "F1": { "fee": 0, "validity": 60, "entries": "M" },
    "F2": { "fee": 0, "validity": 60, "entries": "M" },
    "J1": { "fee": 0, "validity": 60, "entries": "M" },
    "J2": { "fee": 0, "validity": 60, "entries": "M" },
    "H1B": { "fee": 0, "validity": 60, "entries": "M" },
    "H4": { "fee": 0, "validity": 60, "entries": "M" },
    "L1": { "fee": 0, "validity": 60, "entries": "M" },
    "L2": { "fee": 0, "validity": 60, "entries": "M" }
  }
}
```

---

## 4. MODULE API

### Core Functions

```javascript
/**
 * Main entry - get complete visa requirements
 */
function getVisaRequirements(nationality, visaType) {
  return {
    nationality,
    visaType,
    destination: "US",
    
    // Visa waiver check
    visaWaiverEligible: isVisaWaiverEligible(nationality),
    
    // Document checklist
    documents: getDocumentChecklist(visaType),
    
    // Fees
    fees: {
      base: getBaseFee(visaType),
      sevis: getSevisFee(visaType),
      reciprocity: getReciprocityFee(nationality, visaType),
      total: calculateTotalFee(nationality, visaType)
    },
    
    // Validity
    validity: {
      months: getValidityMonths(nationality, visaType),
      entries: getEntries(nationality, visaType)
    },
    
    // Additional info
    photoRequirements: getPhotoRequirements(),
    sixMonthClubExempt: isSixMonthClubMember(nationality),
    dependentVisa: getDependentVisa(visaType),
    petitionRequired: isPetitionRequired(visaType),
    
    // Sources
    sources: getSources(nationality, visaType),
    
    // Legal
    disclaimer: getDisclaimer(),
    lastUpdated: getLastUpdated(nationality, visaType)
  };
}

/**
 * Check VWP eligibility
 */
function isVisaWaiverEligible(nationality) { ... }

/**
 * Get document checklist for visa type
 */
function getDocumentChecklist(visaType) { ... }

/**
 * Get all fees
 */
function getBaseFee(visaType) { ... }
function getSevisFee(visaType) { ... }
function getReciprocityFee(nationality, visaType) { ... }
function calculateTotalFee(nationality, visaType) { ... }

/**
 * Get validity info
 */
function getValidityMonths(nationality, visaType) { ... }
function getEntries(nationality, visaType) { ... }

/**
 * Additional checks
 */
function isSixMonthClubMember(nationality) { ... }
function getDependentVisa(visaType) { ... }
function isPetitionRequired(visaType) { ... }
function getPhotoRequirements() { ... }

/**
 * Sources and metadata
 */
function getSources(nationality, visaType) { ... }
function getDisclaimer() { ... }
function getLastUpdated(nationality, visaType) { ... }
```

### Usage Example
```javascript
import { getVisaRequirements } from './visa-validator';

const result = getVisaRequirements("IN", "F1");

// Result:
{
  nationality: "IN",
  visaType: "F1",
  destination: "US",
  visaWaiverEligible: false,
  documents: {
    required: [
      { id: "passport", name: "Valid Passport", ... },
      { id: "ds160", name: "DS-160 Confirmation", ... },
      { id: "i20", name: "Form I-20", ... },
      { id: "sevis_receipt", name: "SEVIS Fee Receipt", ... },
      ...
    ],
    supporting: [ ... ]
  },
  fees: {
    base: 185,
    sevis: 350,
    reciprocity: 0,
    total: 535
  },
  validity: {
    months: 60,
    entries: "M"
  },
  photoRequirements: { ... },
  sixMonthClubExempt: false,
  dependentVisa: "F2",
  petitionRequired: false,
  sources: { ... },
  disclaimer: "...",
  lastUpdated: "2026-02-16"
}
```

---

## 5. FILE STRUCTURE (Final)

```
visa-validator/
│
├── docs/
│   ├── DISCOVERY.md
│   ├── ARCHITECTURE.md          # This file
│   ├── SCOPE_MATRIX.md
│   └── INTEGRATION.md           # How to integrate
│
├── src/
│   ├── index.js                 # Entry point
│   ├── visa-validator.js        # Main module
│   └── utils.js                 # Helper functions
│
├── data/
│   ├── meta/
│   │   ├── countries.json
│   │   ├── visa_types.json
│   │   ├── visa_waiver.json
│   │   ├── six_month_club.json
│   │   └── photo_specs.json
│   ├── requirements/            # ~40 files
│   └── reciprocity/             # ~200 files
│
├── scripts/
│   ├── scrape_reciprocity.py    # Scrape travel.state.gov
│   ├── scrape_requirements.py   # Scrape visa requirements
│   └── validate_data.py         # Data validation
│
├── dist/
│   ├── visa-validator.js
│   └── visa-validator.min.js
│
└── tests/
    └── visa-validator.test.js
```

---

## 6. TECH STACK

| Component | Technology |
|-----------|------------|
| Module | JavaScript (ES6) |
| Data | JSON files |
| Scraper | Python + BeautifulSoup |
| Build | esbuild or rollup |
| Tests | Jest |

---

## 7. DATA PIPELINE

### Scraping Strategy
```
Source: travel.state.gov

1. Reciprocity Tables
   URL: /us-visas/Visa-Reciprocity-and-Civil-Documents-by-Country/{COUNTRY}.html
   Extract: fee, validity, entries per visa type
   
2. Visa Requirements
   URL: /us-visas/{category}/{visa-type}.html
   Extract: required docs, supporting docs, process
   
3. Visa Waiver Program
   URL: /us-visas/tourism-visit/visa-waiver-program.html
   Extract: eligible countries, restrictions
```

### Update Strategy
- Weekly cron checks for changes
- Diff against existing data
- Alert on changes → manual review → merge

---

## 8. IMPLEMENTATION PLAN

| Step | Task | Time |
|------|------|------|
| 1 | Build reciprocity scraper | 30 min |
| 2 | Scrape all 200 countries | 30 min |
| 3 | Build requirements scraper | 30 min |
| 4 | Scrape all ~40 visa types | 30 min |
| 5 | Create meta files (VWP, 6-month club, etc.) | 30 min |
| 6 | Build JS module | 2 hours |
| 7 | Test + validate | 1 hour |
| 8 | Documentation | 30 min |
| **Total** | | **~6 hours** |

---

## 9. INTEGRATION

### For dummyvisaticket.com

```javascript
// Option 1: Direct import
import { getVisaRequirements } from './lib/visa-validator';

// Option 2: NPM package (future)
import { getVisaRequirements } from 'visa-validator';

// Usage in checkout
const handleVisaCheck = (nationality, visaType) => {
  const requirements = getVisaRequirements(nationality, visaType);
  displayChecklist(requirements);
};
```

---

## 10. APPROVED ✅

- [x] Scope confirmed (complete coverage)
- [x] Architecture approved
- [x] Tech stack approved
- [x] Timeline: ~6 hours

**Next step:** Implementation (scraping)

---

*Architecture by Cookie 🍪 | 2026-02-16*


---

## 11. INTEGRATION STACK (dummyvisaticket.com)

**Added:** 2026-02-17 — Based on actual production codebase review.

### Host App Tech Stack
| Component | Version | Notes |
|-----------|---------|-------|
| Next.js | 15.2.6 | Turbopack, App Router |
| React | 19.x | |
| TypeScript | strict | |
| Tailwind CSS | 4.x | PostCSS plugin |
| Apollo Client | 3.13 | GraphQL to Strapi backend |
| Stripe | 18.x | Payments |
| Zod | 3.24 | Schema validation |
| React Hook Form | 7.55 | Form handling |
| Zustand | 5.x | State management |
| @react-pdf/renderer | 4.3 | PDF generation |
| Amadeus | 5.1 | Flight search |

### Integration Implications

**Module must be TypeScript** — not plain JS. Match the host app's strict TS config.

**Validation with Zod:**
```typescript
import { z } from 'zod';

export const VisaQuerySchema = z.object({
  nationality: z.string().length(2), // ISO 3166-1 alpha-2
  visaType: z.string().min(1),
});

export type VisaQuery = z.infer<typeof VisaQuerySchema>;
```

**State with Zustand (optional):**
```typescript
import { create } from 'zustand';

interface VisaStore {
  nationality: string | null;
  visaType: string | null;
  requirements: VisaRequirements | null;
  setQuery: (nationality: string, visaType: string) => void;
}
```

**Styling with Tailwind** — any UI components use Tailwind classes, no CSS modules.

**PDF generation** — can leverage existing @react-pdf/renderer for downloadable checklists.

### Revised File Structure
```
visa-validator/
├── src/
│   ├── index.ts                 # Entry point (exports all)
│   ├── visa-validator.ts        # Core logic
│   ├── types.ts                 # TypeScript types
│   ├── schemas.ts               # Zod schemas
│   └── utils.ts                 # Helpers
├── data/                        # Static JSON (imported at build time)
├── scripts/                     # Python scrapers (dev only)
└── __tests__/                   # Jest/Vitest tests
```

### Integration Pattern
```typescript
// In dummyvisaticket checkout page:
import { getVisaRequirements } from '@/lib/visa-validator';
import { VisaQuerySchema } from '@/lib/visa-validator/schemas';

// Validate input
const query = VisaQuerySchema.parse({ nationality: 'IN', visaType: 'B1B2' });

// Get requirements
const result = getVisaRequirements(query.nationality, query.visaType);

// Render with Tailwind components
```
