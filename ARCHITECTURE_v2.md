# Visa Document Validator — Architecture Design v2

**Status:** FINAL  
**Version:** 2.0 (updated 2026-02-17)  
**Owner:** Cookie 🍪

---

## 1. CONTEXT

**What:** Plug-and-play TypeScript module for dummyvisaticket.com  
**Integration point:** Checkout flow — user selects nationality + visa type → returns complete checklist  
**Scope MVP:** ALL US visa types (~40) × ALL countries (~200)  
**Scope Future:** US + UK + EU (Schengen) + Canada + Australia

---

## 2. HOST APP STACK

| Component | Version |
|-----------|---------|
| Next.js | 15.2.6 (App Router, Turbopack) |
| TypeScript | strict, ES2017 |
| React | 19.x |
| Tailwind CSS | 4.x |
| Zod | 3.24 |
| Zustand | 5.x |
| React Hook Form | 7.55 |
| Apollo/Strapi | GraphQL backend |
| Stripe | 18.x |
| @react-pdf/renderer | 4.3 |
| Hosting | Vercel |

---

## 3. DATA ARCHITECTURE

### File Structure (destination-agnostic)
```
src/lib/visa-validator/
├── index.ts                    # Public API exports
├── types.ts                    # TypeScript interfaces
├── schemas.ts                  # Zod validation schemas
├── validator.ts                # Core logic
├── utils/
│   ├── fees.ts                 # Fee calculations
│   ├── validity.ts             # Validity lookups
│   └── documents.ts            # Document checklist logic
│
├── data/
│   ├── shared/
│   │   ├── countries.json      # ISO 3166-1 (~200 countries)
│   │   └── photo_specs.json    # Photo requirements (varies by destination)
│   │
│   ├── US/
│   │   ├── visa_types.json     # All ~40 US visa types
│   │   ├── visa_waiver.json    # VWP countries + restrictions
│   │   ├── six_month_club.json # Passport validity exemptions
│   │   ├── reciprocity.json    # ALL countries × ALL visa types (single flat file)
│   │   └── requirements/       # Document checklists per visa type
│   │       ├── B1B2.json
│   │       ├── F1.json
│   │       └── ... (~40 files)
│   │
│   ├── UK/                     # Future
│   │   ├── visa_types.json
│   │   ├── reciprocity.json
│   │   └── requirements/
│   │
│   ├── EU/                     # Future (Schengen)
│   │   └── ...
│   │
│   ├── CA/                     # Future
│   │   └── ...
│   │
│   └── AU/                     # Future
│       └── ...
│
└── __tests__/
    └── validator.test.ts
```

### Reciprocity: Single Flat File (per destination)
```json
// data/US/reciprocity.json
{
  "lastUpdated": "2026-02-17",
  "source": "travel.state.gov",
  "data": {
    "IN": {
      "B1B2": { "fee": 0, "validityMonths": 120, "entries": "M" },
      "F1": { "fee": 0, "validityMonths": 60, "entries": "M" },
      "H1B": { "fee": 0, "validityMonths": 60, "entries": "M" }
    },
    "CN": {
      "B1B2": { "fee": 185, "validityMonths": 120, "entries": "M" },
      "F1": { "fee": 0, "validityMonths": 60, "entries": "M" }
    },
    "BG": {
      "B1B2": { "fee": 0, "validityMonths": 120, "entries": "M" }
    }
  }
}
```

**Why flat:** One file instead of 200. Faster lookups. Scales to multiple destinations without file explosion.

---

## 4. MODULE API

### Core Function (destination-aware)
```typescript
type Destination = 'US' | 'UK' | 'EU' | 'CA' | 'AU';

interface VisaQuery {
  nationality: string;      // ISO 3166-1 alpha-2
  visaType: string;          // e.g., "B1B2", "F1"
  destination: Destination;  // defaults to "US" for MVP
}

interface VisaRequirements {
  query: VisaQuery;
  visaWaiverEligible: boolean;
  documents: {
    required: Document[];
    supporting: Document[];
  };
  fees: {
    base: number;
    reciprocity: number;
    sevis: number | null;
    total: number;
  };
  validity: {
    months: number;
    entries: '1' | '2' | 'M';
  };
  photoRequirements: PhotoSpecs;
  sixMonthClubExempt: boolean;
  dependentVisa: string | null;
  petitionRequired: boolean;
  commonMistakes: string[];
  interviewTips: string[];
  sources: string[];
  disclaimer: string;
  lastUpdated: string;
}
```

### Result Type Pattern
```typescript
type VisaResult<T> =
  | { ok: true; value: T }
  | { ok: false; error: { type: 'VALIDATION' | 'NOT_FOUND' | 'UNSUPPORTED'; message: string } };

function getVisaRequirements(query: VisaQuery): VisaResult<VisaRequirements>;
function isVisaWaiverEligible(nationality: string, destination?: Destination): boolean;
function getDocumentChecklist(visaType: string, destination?: Destination): VisaResult<DocumentChecklist>;
function getFees(nationality: string, visaType: string, destination?: Destination): VisaResult<FeeBreakdown>;
```

### Zod Schemas
```typescript
import { z } from 'zod';

export const VisaQuerySchema = z.object({
  nationality: z.string().length(2).toUpperCase(),
  visaType: z.string().min(1).max(10),
  destination: z.enum(['US', 'UK', 'EU', 'CA', 'AU']).default('US'),
});

export type VisaQueryInput = z.infer<typeof VisaQuerySchema>;
```

---

## 5. INTEGRATION

```typescript
// In dummyvisaticket checkout:
import { getVisaRequirements, VisaQuerySchema } from '@/lib/visa-validator';

const handleCheck = (nationality: string, visaType: string) => {
  const query = VisaQuerySchema.parse({ nationality, visaType, destination: 'US' });
  const result = getVisaRequirements(query);
  
  if (!result.ok) {
    showError(result.error.message);
    return;
  }
  
  displayChecklist(result.value);
};
```

---

## 6. DATA PIPELINE

### Scraping (dev-only, Python + BeautifulSoup)
```
travel.state.gov
    │
    ▼
scrape_reciprocity.py (with stealth: delays, UA rotation)
    │
    ▼
Raw HTML → parse tables → validate
    │
    ▼
data/US/reciprocity.json (single flat file)
```

### Change Detection (hash-based)
```python
import hashlib, json

def check_for_changes(country_code: str, new_data: dict) -> bool:
    current = load_existing_reciprocity()
    old_hash = hashlib.sha256(json.dumps(current.get(country_code, {}), sort_keys=True).encode()).hexdigest()
    new_hash = hashlib.sha256(json.dumps(new_data, sort_keys=True).encode()).hexdigest()
    return old_hash != new_hash
```

### Update Strategy
- Weekly cron scrapes travel.state.gov
- Hash comparison per country
- Changes detected → alert for manual review
- No auto-deploy of data changes

---

## 7. TECH STACK

| Component | Technology |
|-----------|------------|
| Module | TypeScript (strict) |
| Validation | Zod |
| Data | Static JSON (build-time imports) |
| Scraper | Python + BeautifulSoup |
| Tests | Vitest |
| Hosting | Vercel (with host app) |

---

## 8. BUNDLE STRATEGY

**MVP approach:** Static imports. All JSON bundled at build time.

**Estimated sizes:**
| File | Raw | Gzipped |
|------|-----|---------|
| countries.json | ~10KB | ~3KB |
| visa_types.json | ~15KB | ~5KB |
| reciprocity.json | ~500KB | ~100KB |
| requirements/ (all) | ~200KB | ~50KB |
| meta files | ~5KB | ~2KB |
| **Total** | **~730KB** | **~160KB** |

**Optimization (if needed later):**
- Dynamic import reciprocity.json only when user selects nationality
- Move to API route with edge caching
- Not needed for MVP — 160KB gzipped is acceptable for a checkout page

---

## 9. IMPLEMENTATION PLAN

| Step | Task | Est. |
|------|------|------|
| 1 | Create meta data files (countries, visa_types, VWP, 6-month club, photo specs) | 1h |
| 2 | Build reciprocity scraper (Python) | 1h |
| 3 | Run scraper → generate reciprocity.json | 30-60min |
| 4 | Write requirements JSONs (all ~40 visa types) | 2-3h |
| 5 | Build TypeScript module (validator + types + schemas) | 2h |
| 6 | Tests + validation | 1h |
| 7 | Integration docs | 30min |
| **Total** | | **~8-9h** |

---

## 10. FUTURE EXPANSION

| Destination | Data Source | Effort |
|-------------|------------|--------|
| UK | gov.uk/standard-visitor | Medium |
| EU (Schengen) | ec.europa.eu + per-country | High (27 countries) |
| Canada | canada.ca/immigration | Medium |
| Australia | immi.homeaffairs.gov.au | Medium |

Each destination = new data folder + scraper. Module API stays the same (`destination` parameter).

---

## 11. APPROVED ✅

- [x] Destination-agnostic API (future-proofed)
- [x] Flat reciprocity file (not 200 separate files)
- [x] Result type error handling
- [x] Zod validation
- [x] Hash-based change detection
- [x] TypeScript strict mode
- [x] Compatible with Next.js 15 + Vercel

**Next step:** Implementation

---

*Architecture v2 by Cookie 🍪 | 2026-02-17*

---

## 12. REVIEW PROTOCOL

**Mandatory for every design phase:**

1. Cookie designs the codebase/approach
2. Spawn review panel:
   - **Cookie** (Opus) — architecture lead
   - **Kimi K2.5** — reviewer #1
   - **Claude Sonnet 4.5** — reviewer #2
3. All 3 must agree before proceeding to implementation
4. Disagreements → escalate to Martin for final call

**No implementation without unanimous agreement.**
