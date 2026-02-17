# Visa Requirements Research Findings
**Date:** 2026-02-16

## Key Question: Are requirements different by nationality/location?

### Answer: PARTIALLY

**What's SAME for everyone:**
- Core document checklist (passport, forms, photos, supporting docs)
- Process (online application → biometrics → interview)
- Eligibility criteria (intent to return, financial ability, etc.)

**What DIFFERS by nationality:**
1. **Whether you need a visa at all** (Visa Waiver, ETA, or full visa)
2. **Fees** (US has "reciprocity fees" per nationality)
3. **Validity period** (varies by nationality)
4. **Wait times** (varies by embassy location)

---

## US B1/B2 Visa

### Nationality Tiers

| Tier | Who | What They Need |
|------|-----|----------------|
| **Visa Waiver (ESTA)** | 41 countries (EU, UK, Japan, Australia, etc.) | Just ESTA ($21), no visa |
| **Visa Required** | Everyone else | Full B1/B2 visa process |

### Core Documents (same for all who need visa)
1. Valid passport (6+ months validity)
2. DS-160 online form confirmation
3. Photo (per specifications)
4. MRV fee receipt ($185)
5. Appointment confirmation

### Supporting Documents (prove ties to home country)
- Employment letter
- Bank statements
- Property ownership
- Family ties documentation
- Travel history
- Invitation letter (if applicable)

### What Varies by Nationality
- **Reciprocity fee**: Additional fee beyond $185 (country-specific)
- **Visa validity**: Could be 1 year, 5 years, or 10 years depending on nationality
- **Number of entries**: Single vs multiple

### Location of Application
From State Dept: *"You should generally schedule an appointment at the U.S. Embassy or Consulate in the country where you live. You may schedule at another Embassy or Consulate where you will be present but it may be more difficult to demonstrate your qualifications outside your home country."*

**Key insight:** An Indian applying from UAE uses the SAME documents as an Indian applying from India. BUT:
- Wait times differ (UAE vs India embassies)
- Proving "ties to home country" may be harder from third country
- Same nationality-based fee applies regardless of location

---

## UK Standard Visitor Visa

### Nationality Tiers

| Tier | Who | What They Need |
|------|-----|----------------|
| **No visa/ETA needed** | EU, US, Canada, Australia, etc. | Nothing, just passport |
| **ETA required** | GCC countries, some others | ETA (£10) |
| **Visa required** | India, China, most African/Asian countries | Full visa (£127+) |

### Core Documents (same for all who need visa)
1. Valid passport with blank page
2. Online application
3. Travel dates/itinerary
4. Accommodation details
5. Financial evidence (bank statements, payslips)
6. Employment details
7. Sponsor details (if funded by someone else)

### Additional Requirements (specific situations)
- TB test (if visiting 6+ months, from certain countries)
- Study documentation (if studying)
- Medical documentation (if for treatment)

### Fees (same for all nationalities)
- Standard: £127
- Long-term 2yr: £475
- Long-term 5yr: £848
- Long-term 10yr: £1,059

---

## Data Model Implications

```
NATIONALITY_TIERS = {
    "US_DESTINATION": {
        "visa_waiver": ["DE", "FR", "UK", "JP", ...],  # 41 countries
        "visa_required": ["IN", "CN", "NG", ...]       # everyone else
    },
    "UK_DESTINATION": {
        "visa_free": ["US", "CA", "AU", ...],
        "eta_required": ["AE", "SA", "QA", ...],
        "visa_required": ["IN", "CN", "PK", ...]
    }
}

REQUIREMENTS = {
    "US_B1B2": {
        "documents": [...],  # same for everyone
        "fees": {
            "base": 185,
            "reciprocity": {  # varies by nationality
                "IN": 0,
                "RU": 303,
                ...
            }
        }
    },
    "UK_VISITOR": {
        "documents": [...],  # same for everyone
        "fees": {
            "standard": 127,
            "long_term_2yr": 475,
            ...
        }
    }
}
```

---

## MVP Scope Recommendation

### Phase 1: Start Simple
1. **US B1/B2** destination only
2. **Top 10 nationalities** that need visas (India, China, Philippines, etc.)
3. **Standard tourist/business** purpose only
4. **Single checklist** per destination (same for all nationalities)

### Phase 2: Add Complexity
1. Add UK Standard Visitor
2. Add Schengen (complex - multiple entry points)
3. Add nationality-specific fees/validity info
4. Add third-country application considerations

---

## Sources
- travel.state.gov (US State Department)
- gov.uk (UK Home Office)
- uscis.gov (US Citizenship and Immigration Services)
