# Visa Validator — Review Brief for QA

## Project Goal
Plug-and-play TypeScript module for dummyvisaticket.com (a dummy flight ticket booking site).
User selects nationality + visa type → gets complete checklist of documents, fees, validity.
Upsell opportunity in checkout flow.

## Host App Stack (dummyvisaticket.com)
- **Framework:** Next.js 15.2.6 (App Router, Turbopack)
- **Language:** TypeScript (strict mode, ES2017 target)
- **React:** 19.x
- **Styling:** Tailwind CSS v4
- **State:** Zustand 5.x
- **Validation:** Zod 3.24
- **Forms:** React Hook Form 7.55
- **GraphQL:** Apollo Client 3.13 → Strapi backend
- **Payments:** Stripe 18.x
- **Email:** Resend
- **Flights:** Migrating from Amadeus to Duffel API
- **PDF:** @react-pdf/renderer 4.3
- **Hosting:** Vercel
- **Path alias:** @/* → ./src/*

## Visa Validator Module Design

### Scope
- ALL US nonimmigrant visa types (~40) × ALL countries (~200)
- ~8,000 country×visa combinations
- Static JSON data, no backend needed

### Data Architecture
```
data/
├── meta/
│   ├── countries.json         # ~200 countries (ISO 3166-1)
│   ├── visa_types.json        # ~40 visa types with base fees, SEVIS fees, petition requirements
│   ├── visa_waiver.json       # VWP countries + restrictions
│   ├── six_month_club.json    # Passport validity exemptions
│   └── photo_specs.json       # US visa photo requirements
├── requirements/              # Document checklists per visa type (~40 files)
│   ├── B1B2.json
│   ├── F1.json
│   └── ...
└── reciprocity/               # Per-country fees/validity (~200 files)
    ├── IN.json
    ├── CN.json
    └── ...
```

### Module API
```typescript
getVisaRequirements(nationality: string, visaType: string): VisaRequirements
// Returns: documents, fees (base+reciprocity+SEVIS), validity, sources, disclaimers

isVisaWaiverEligible(nationality: string): boolean
getDocumentChecklist(visaType: string): DocumentChecklist
getFees(nationality: string, visaType: string): FeeBreakdown
getValidity(nationality: string, visaType: string): ValidityInfo
```

### Integration Pattern
```typescript
// In dummyvisaticket checkout:
import { getVisaRequirements } from '@/lib/visa-validator';
const result = getVisaRequirements('IN', 'B1B2');
```

### Data Source
- travel.state.gov (official US State Dept)
- Reciprocity pages: /Visa-Reciprocity-and-Civil-Documents-by-Country/{COUNTRY}.html
- Scraped with Python + BeautifulSoup (dev-only tool)
- Weekly cron to check for changes

### Tech Decisions
- TypeScript module (not plain JS)
- Zod schemas for input validation
- Static JSON imported at build time (Next.js bundler handles it)
- No runtime API calls needed
- ~750KB total data size
- Python scraper for data collection (dev tooling only)

### Key Data Points Per Query
- VWP eligibility check
- Document checklist (required + supporting)
- Fee breakdown (base MRV + reciprocity + SEVIS)
- Visa validity (months) and entries (1/2/M)
- Six Month Club passport exemption
- Photo specifications
- Common mistakes + interview tips
- Dependent visa info
- Official source URLs

## Questions for Reviewer
1. Is the data architecture sound for ~8,000 combinations?
2. Should JSON be bundled at build time or loaded dynamically?
3. Any concerns with ~750KB of static JSON in a Next.js bundle?
4. Is the scraper design (Python + BeautifulSoup) appropriate for travel.state.gov?
5. Are there missing data points or edge cases?
6. Should this be a separate npm package or just a /lib folder in the Next.js app?
7. Any TypeScript patterns or Next.js best practices we should follow?
