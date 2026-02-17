# Visa Document Validator

Validation tool that checks if a visa application is complete and correct before submission.

## Status: Research Phase

## Concept

- **Input:** Visa type + Destination + Nationality + Residence country
- **Output:** Required documents checklist + validation

## Key Questions (Researching)

1. Do requirements differ by nationality?
2. Does country of residence (where you apply) matter?
3. Are there nationality "tiers" with different requirements?

## Target Visas (MVP)

- US B1/B2 (Tourist/Business)
- UK Standard Visitor Visa

## Architecture (Planned)

```
data/
├── requirements/
│   ├── us-b1b2.json      # US visa requirements
│   └── uk-visitor.json   # UK visa requirements
├── nationality-tiers.json # If requirements vary by nationality
src/
├── validator.py          # Core validation logic
├── api.py               # Web API
└── ui/                  # Frontend
```

## Monetization

| Model | Price |
|-------|-------|
| Per validation | $10-15 |
| Bundle with dummy ticket | $20-25 |

## Links

- Parent project: dummyvisaticket.com
- Money ideas: memory/projects/money-ideas.md

---
*Created: 2026-02-16*
