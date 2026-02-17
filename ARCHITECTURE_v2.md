# Visa Document Validator — Architecture Design v2

**Status:** APPROVED  
**Version:** 2.0 (updated 2026-02-17)  
**Owner:** Cookie 🍪  
**Reviewed by:** Cookie (Opus), Kimi K2.5, Claude Sonnet 4.5

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
| Flights | Duffel API (migrating from Amadeus) |
| Hosting | Vercel |

---

## 3. DATA ARCHITECTURE

### File Structure (destination-agnostic)
```
src/lib/visa-validator/
├── index.ts                    # Public API exports
├── types.ts                    # TypeScript interfaces + branded types
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
│   └── US/
│       ├── visa_types.json     # All ~40 US visa types
│       ├── visa_waiver.json    # VWP countries + restrictions
│       ├── six_month_club.json # Passport validity exemptions
│       ├── reciprocity.json    # ALL countries × ALL visa types (single flat file)
│       └── requirements/       # Document checklists per visa type
│           ├── B1B2.json
│           ├── F1.json
│           └── ... (~40 files)
│
└── __tests__/
    ├── validator.test.ts
    └── data-integrity.test.ts  # Validates JSON data consistency
```

Future destinations add: `data/UK/`, `data/EU/`, `data/CA/`, `data/AU/`

### Reciprocity: Single Flat File (per destination)
```json
{
  "_meta": {
    "scraped_at": "2026-02-17T08:00:00Z",
    "source": "https://travel.state.gov/...",
    "schema_version": "1.0.0",
    "hash": "abc123..."
  },
  "data": {
    "IN": {
      "B1B2": { "fee": 0, "validityMonths": 120, "entries": "M" },
      "F1": { "fee": 0, "validityMonths": 60, "entries": "M" },
      "H1B": { "fee": 0, "validityMonths": 60, "entries": "M" }
    },
    "CN": {
      "B1B2": { "fee": 185, "validityMonths": 120, "entries": "M" },
      "F1": { "fee": 0, "validityMonths": 60, "entries": "M" }
    }
  }
}
```

---

## 4. MODULE API

### Branded Types
```typescript
type Brand<K, T> = K & { __brand: T };
export type CountryCode = Brand<string, 'CountryCode'>;
export type VisaTypeCode = Brand<string, 'VisaTypeCode'>;
export type Destination = 'US' | 'UK' | 'EU' | 'CA' | 'AU';
```

### Core Function
```typescript
interface VisaQuery {
  nationality: CountryCode;
  visaType: VisaTypeCode;
  destination: Destination;  // REQUIRED — no default
}

type VisaResult<T> =
  | { ok: true; value: T }
  | { ok: false; error: { type: 'VALIDATION' | 'NOT_FOUND' | 'UNSUPPORTED'; message: string } };

function getVisaRequirements(query: VisaQuery): VisaResult<VisaRequirements>;
```

### Zod Schemas (shared with host app forms)
```typescript
import { z } from 'zod';

export const VisaQuerySchema = z.object({
  nationality: z.string().length(2).toUpperCase(),
  visaType: z.string().min(1).max(10),
  destination: z.enum(['US', 'UK', 'EU', 'CA', 'AU']),  // Required
});

export type VisaQueryInput = z.infer<typeof VisaQuerySchema>;
```

---

## 5. BUNDLE STRATEGY

### Dynamic Import (approved by review panel)
```typescript
// ❌ DON'T: Static import on page load
import { getVisaRequirements } from '@/lib/visa-validator';

// ✅ DO: Load on-demand when user interacts
const handleValidatorOpen = async () => {
  const { getVisaRequirements } = await import('@/lib/visa-validator');
  const result = getVisaRequirements(query);
};
```

**Meta files** (countries, visa_types, VWP list) = static import (~50KB, needed for dropdowns)  
**Reciprocity data** (~500KB) = dynamic import on query

### Estimated Sizes
| File | Raw | Gzipped | Load Strategy |
|------|-----|---------|---------------|
| countries.json | ~10KB | ~3KB | Static (dropdowns) |
| visa_types.json | ~15KB | ~5KB | Static (dropdowns) |
| visa_waiver.json | ~2KB | ~1KB | Static |
| six_month_club.json | ~1KB | ~0.5KB | Static |
| reciprocity.json | ~500KB | ~100KB | **Dynamic** |
| requirements/ (all) | ~200KB | ~50KB | **Dynamic** |
| **Initial bundle** | **~28KB** | **~10KB** | |
| **On-demand** | **~700KB** | **~150KB** | |

---

## 6. INTEGRATION

```typescript
// In dummyvisaticket checkout:
import { VisaQuerySchema } from '@/lib/visa-validator/schemas';

const handleCheck = async (nationality: string, visaType: string) => {
  const query = VisaQuerySchema.parse({ nationality, visaType, destination: 'US' });
  const { getVisaRequirements } = await import('@/lib/visa-validator');
  const result = getVisaRequirements(query);
  
  if (!result.ok) {
    showError(result.error.message);
    return;
  }
  
  displayChecklist(result.value);
};
```

---

## 7. DATA PIPELINE

### Scraping (dev-only, Python + BeautifulSoup)
- Stealth: delays, UA rotation, respect robots.txt
- **Schema validation** on every scrape (Zod schemas validate output before writing)
- **Hash-based change detection** per country
- **Metadata** in every JSON file (scraped_at, source, hash)

### Scraper Validation
```python
# After scraping, validate against schema
def validate_scraped_data(data: dict) -> bool:
    for country_code, visas in data.items():
        assert len(country_code) == 2
        for visa_code, info in visas.items():
            assert isinstance(info['fee'], int) and info['fee'] >= 0
            assert isinstance(info['validityMonths'], int) and info['validityMonths'] > 0
            assert info['entries'] in ['1', '2', 'M']
    return True
```

### Data Integrity Tests
```typescript
// __tests__/data-integrity.test.ts
describe('Visa Data Integrity', () => {
  it('all reciprocity countries exist in countries.json', () => { ... });
  it('all visa types have requirements files', () => { ... });
  it('all fees are non-negative integers', () => { ... });
  it('all entries are 1, 2, or M', () => { ... });
});
```

### Update Strategy
- Weekly cron checks travel.state.gov
- Hash comparison → only flag changes
- Human review before merge
- CI validates all JSON on every PR

---

## 8. TECH STACK

| Component | Technology |
|-----------|------------|
| Module | TypeScript (strict) |
| Validation | Zod |
| Types | Branded types (CountryCode, VisaTypeCode) |
| Error handling | Discriminated union Result type |
| Data | Static JSON (dynamic import) |
| Scraper | Python + BeautifulSoup |
| Tests | Vitest + data integrity suite |
| Hosting | Vercel (with host app) |

---

## 9. IMPLEMENTATION PLAN

| Step | Task | Est. |
|------|------|------|
| 1 | Create meta data files (countries, visa_types, VWP, 6-month club, photo specs) | 1h |
| 2 | Build reciprocity scraper (Python + validation) | 1.5h |
| 3 | Run scraper → generate reciprocity.json | 30-60min |
| 4 | Write requirements JSONs (all ~40 visa types) | 2-3h |
| 5 | Build TypeScript module (validator + types + schemas + branded types) | 2h |
| 6 | Data integrity tests | 30min |
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

Each destination = new `data/{DEST}/` folder + scraper. Module API unchanged.

---

## 11. REVIEW PROTOCOL

**Mandatory for every design phase:**

1. Cookie designs the codebase/approach
2. Review panel: **Cookie** (Opus) + **Kimi K2.5** + **Claude Sonnet 4.5**
3. All 3 must agree before proceeding
4. Disagreements → escalate to Martin

---

## 12. APPROVED ✅

**Review panel consensus (2026-02-17):**
- [x] Flat reciprocity file
- [x] Dynamic import for heavy data
- [x] Destination parameter required (no default)
- [x] Branded types for CountryCode/VisaTypeCode
- [x] Schema validation in scraper
- [x] Metadata in JSON files
- [x] Data integrity tests
- [x] Result type error handling

**Rejected for MVP (may add later):**
- Processing times, 214(b) risk indicators, PP 10043
- Playwright scraper fallback
- neverthrow library

**Next step:** Implementation

---

*Architecture v2 by Cookie 🍪 | Reviewed: Kimi K2.5, Sonnet 4.5 | 2026-02-17*
