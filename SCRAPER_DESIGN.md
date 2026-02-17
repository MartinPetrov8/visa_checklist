# Visa Validator — Scraper Design

**Status:** DRAFT — Pending QA Review  
**Created:** 2026-02-17

---

## 1. OVERVIEW

**Goal:** Scrape travel.state.gov for visa reciprocity data (fees, validity, entries) for all ~200 countries and ~40 visa types.

**Stealth Requirements:**
- Random delays between requests
- Rotating User-Agent headers
- Respect robots.txt
- No parallel requests (sequential only)
- Session persistence (resume on failure)

---

## 2. TARGET PAGES

### 2.1 Country Reciprocity Pages
```
Base URL: https://travel.state.gov/content/travel/en/us-visas/Visa-Reciprocity-and-Civil-Documents-by-Country/{COUNTRY_NAME}.html

Examples:
- .../India.html
- .../China.html
- .../United-Kingdom.html
- .../Korea-South.html
```

**Data to extract:**
- Reciprocity fee per visa type
- Validity period (months)
- Number of entries (1, 2, M)

### 2.2 Visa Requirements Pages
```
Base URLs:
- /us-visas/tourism-visit/visitor.html (B1/B2)
- /us-visas/study/student-visa.html (F1)
- /us-visas/study/exchange.html (J1)
- /us-visas/employment/temporary-worker-visas.html (H/L/O/P)
```

**Data to extract:**
- Required documents list
- Process steps
- Fee information

### 2.3 Reference Pages
```
- /us-visas/tourism-visit/visa-waiver-program.html (VWP countries)
- /us-visas/visa-information-resources/fees/fees-visa-services.html (base fees)
```

---

## 3. SCRAPER ARCHITECTURE

```
┌─────────────────────────────────────────────────────────────────┐
│                         SCRAPER                                  │
│                                                                  │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐           │
│  │   Config    │   │   Fetcher   │   │   Parser    │           │
│  │  (delays,   │──▶│  (stealth   │──▶│  (extract   │           │
│  │   headers)  │   │   requests) │   │   data)     │           │
│  └─────────────┘   └─────────────┘   └─────────────┘           │
│                           │                  │                   │
│                           ▼                  ▼                   │
│                    ┌─────────────┐   ┌─────────────┐           │
│                    │   Cache     │   │   Output    │           │
│                    │  (resume)   │   │   (JSON)    │           │
│                    └─────────────┘   └─────────────┘           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. STEALTH CONFIGURATION

### 4.1 User-Agent Rotation
```python
USER_AGENTS = [
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36",
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:123.0) Gecko/20100101 Firefox/123.0",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.2 Safari/605.1.15",
    "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/122.0.0.0 Safari/537.36",
]
```

### 4.2 Request Headers
```python
HEADERS = {
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8",
    "Accept-Language": "en-US,en;q=0.9",
    "Accept-Encoding": "gzip, deflate, br",
    "Connection": "keep-alive",
    "Upgrade-Insecure-Requests": "1",
    "Sec-Fetch-Dest": "document",
    "Sec-Fetch-Mode": "navigate",
    "Sec-Fetch-Site": "none",
    "Sec-Fetch-User": "?1",
    "Cache-Control": "max-age=0",
}
```

### 4.3 Delay Strategy
```python
DELAY_CONFIG = {
    "min_delay": 3,        # Minimum seconds between requests
    "max_delay": 8,        # Maximum seconds between requests
    "error_delay": 30,     # Delay after error
    "batch_delay": 60,     # Delay every 20 requests
    "batch_size": 20,      # Requests before batch delay
}
```

---

## 5. DATA EXTRACTION

### 5.1 Reciprocity Table Parser

**HTML Structure (observed):**
```html
<table class="reciprocity-table">
  <tr>
    <td>B1/B2</td>
    <td>$0</td>
    <td>120 months</td>
    <td>M</td>
  </tr>
  ...
</table>
```

**Parser Logic:**
```python
def parse_reciprocity_table(html: str) -> dict:
    """
    Extract visa reciprocity data from country page.
    
    Returns:
        {
            "B1B2": {"fee": 0, "validity_months": 120, "entries": "M"},
            "F1": {"fee": 0, "validity_months": 60, "entries": "M"},
            ...
        }
    """
    soup = BeautifulSoup(html, 'html.parser')
    
    # Find reciprocity table/section
    # Multiple possible selectors (site structure varies)
    selectors = [
        'table.reciprocity-table',
        '#visa-classifications table',
        '.visa-reciprocity-data',
    ]
    
    results = {}
    
    # Parse each visa type row
    for row in table.find_all('tr'):
        cols = row.find_all('td')
        if len(cols) >= 4:
            visa_type = normalize_visa_code(cols[0].text.strip())
            fee = parse_fee(cols[1].text.strip())
            validity = parse_validity(cols[2].text.strip())
            entries = parse_entries(cols[3].text.strip())
            
            results[visa_type] = {
                "fee": fee,
                "validity_months": validity,
                "entries": entries
            }
    
    return results
```

### 5.2 Helper Functions

```python
def normalize_visa_code(raw: str) -> str:
    """
    Normalize visa code to standard format.
    
    "B-1/B-2" -> "B1B2"
    "F-1" -> "F1"
    "H-1B" -> "H1B"
    """
    code = raw.upper()
    code = code.replace("-", "").replace("/", "").replace(" ", "")
    return code

def parse_fee(raw: str) -> int:
    """
    Parse fee string to integer.
    
    "$160" -> 160
    "No Fee" -> 0
    "N/A" -> None
    """
    if "no fee" in raw.lower() or raw == "0":
        return 0
    
    match = re.search(r'\$?(\d+)', raw)
    if match:
        return int(match.group(1))
    
    return None

def parse_validity(raw: str) -> int:
    """
    Parse validity to months.
    
    "120 months" -> 120
    "10 years" -> 120
    "5 Years" -> 60
    "12 Months" -> 12
    """
    raw = raw.lower().strip()
    
    # Check for months
    match = re.search(r'(\d+)\s*month', raw)
    if match:
        return int(match.group(1))
    
    # Check for years
    match = re.search(r'(\d+)\s*year', raw)
    if match:
        return int(match.group(1)) * 12
    
    return None

def parse_entries(raw: str) -> str:
    """
    Parse entries to standard format.
    
    "Multiple" -> "M"
    "M" -> "M"
    "1" -> "1"
    "One" -> "1"
    "Two" -> "2"
    """
    raw = raw.strip().upper()
    
    if raw in ["M", "MULTIPLE", "MULT"]:
        return "M"
    if raw in ["1", "ONE", "SINGLE"]:
        return "1"
    if raw in ["2", "TWO"]:
        return "2"
    
    return raw
```

---

## 6. COUNTRY LIST

### 6.1 URL Mapping

**Challenge:** Country names in URL don't always match ISO names.

```python
COUNTRY_URL_MAP = {
    "AF": "Afghanistan",
    "AL": "Albania",
    "DZ": "Algeria",
    "AD": "Andorra",
    "AO": "Angola",
    "AG": "Antigua-and-Barbuda",
    "AR": "Argentina",
    "AM": "Armenia",
    "AU": "Australia",
    "AT": "Austria",
    # ... 
    "KR": "Korea-South",
    "KP": "Korea-North",
    "GB": "United-Kingdom",
    "US": None,  # Skip - no US visa for US citizens
    "TW": "Taiwan",
    # ...
}
```

### 6.2 Discovery Strategy

1. Start with known URL mappings
2. Try standard transformations (spaces → hyphens)
3. Log 404s for manual review
4. Build complete mapping incrementally

---

## 7. OUTPUT FORMAT

### 7.1 Per Country File
```json
// data/reciprocity/IN.json
{
  "countryCode": "IN",
  "countryName": "India",
  "lastUpdated": "2026-02-17",
  "source": "https://travel.state.gov/.../India.html",
  "scrapedAt": "2026-02-17T06:30:00Z",
  "visaTypes": {
    "B1B2": { "fee": 0, "validityMonths": 120, "entries": "M" },
    "F1": { "fee": 0, "validityMonths": 60, "entries": "M" },
    "J1": { "fee": 0, "validityMonths": 60, "entries": "M" },
    "H1B": { "fee": 0, "validityMonths": 60, "entries": "M" }
  }
}
```

### 7.2 Scraper State File (for resume)
```json
// .scraper_state.json
{
  "lastRun": "2026-02-17T06:30:00Z",
  "completed": ["AF", "AL", "DZ", ...],
  "failed": ["XX"],
  "remaining": ["YY", "ZZ", ...]
}
```

---

## 8. ERROR HANDLING

| Error | Action |
|-------|--------|
| 404 Not Found | Log, skip country, continue |
| 403 Forbidden | Stop, alert (possible block) |
| 429 Too Many Requests | Wait 5 min, retry |
| 500 Server Error | Wait 30s, retry 3x, then skip |
| Connection Error | Wait 30s, retry 3x |
| Parse Error | Log raw HTML, skip, continue |

---

## 9. VALIDATION

### 9.1 Data Validation Rules
```python
def validate_country_data(data: dict) -> list[str]:
    """Return list of validation errors."""
    errors = []
    
    if not data.get("countryCode"):
        errors.append("Missing country code")
    
    if len(data.get("countryCode", "")) != 2:
        errors.append("Invalid country code format")
    
    if not data.get("visaTypes"):
        errors.append("No visa types found")
    
    for visa, info in data.get("visaTypes", {}).items():
        if info.get("fee") is None:
            errors.append(f"{visa}: Missing fee")
        if info.get("validityMonths") is None:
            errors.append(f"{visa}: Missing validity")
        if info.get("entries") not in ["1", "2", "M", None]:
            errors.append(f"{visa}: Invalid entries value")
    
    return errors
```

### 9.2 Post-Scrape Validation
```python
def validate_all_data():
    """Validate all scraped data."""
    
    issues = []
    
    # Check all expected countries exist
    for code in EXPECTED_COUNTRIES:
        path = f"data/reciprocity/{code}.json"
        if not os.path.exists(path):
            issues.append(f"Missing: {code}")
    
    # Validate each file
    for path in glob("data/reciprocity/*.json"):
        data = json.load(open(path))
        errors = validate_country_data(data)
        if errors:
            issues.append(f"{path}: {errors}")
    
    return issues
```

---

## 10. SCRAPER PSEUDOCODE

```python
#!/usr/bin/env python3
"""
Visa Reciprocity Scraper
Stealth scraper for travel.state.gov
"""

import json
import random
import time
import requests
from bs4 import BeautifulSoup
from pathlib import Path

class VisaScraper:
    def __init__(self):
        self.session = requests.Session()
        self.state_file = Path(".scraper_state.json")
        self.output_dir = Path("data/reciprocity")
        self.output_dir.mkdir(parents=True, exist_ok=True)
        
    def get_random_headers(self) -> dict:
        """Return headers with random User-Agent."""
        headers = HEADERS.copy()
        headers["User-Agent"] = random.choice(USER_AGENTS)
        return headers
    
    def fetch_page(self, url: str) -> str | None:
        """Fetch page with stealth measures."""
        try:
            # Random delay
            delay = random.uniform(
                DELAY_CONFIG["min_delay"],
                DELAY_CONFIG["max_delay"]
            )
            time.sleep(delay)
            
            # Make request
            response = self.session.get(
                url,
                headers=self.get_random_headers(),
                timeout=30
            )
            
            if response.status_code == 200:
                return response.text
            elif response.status_code == 404:
                print(f"404: {url}")
                return None
            elif response.status_code == 403:
                raise Exception("403 Forbidden - Possible block")
            elif response.status_code == 429:
                print("Rate limited, waiting 5 min...")
                time.sleep(300)
                return self.fetch_page(url)  # Retry
            else:
                print(f"Error {response.status_code}: {url}")
                return None
                
        except requests.exceptions.RequestException as e:
            print(f"Request error: {e}")
            return None
    
    def scrape_country(self, country_code: str, country_name: str) -> dict | None:
        """Scrape reciprocity data for one country."""
        url = f"{BASE_URL}/{country_name}.html"
        
        html = self.fetch_page(url)
        if not html:
            return None
        
        # Parse reciprocity table
        visa_data = parse_reciprocity_table(html)
        
        if not visa_data:
            print(f"No data found for {country_code}")
            return None
        
        return {
            "countryCode": country_code,
            "countryName": country_name.replace("-", " "),
            "lastUpdated": datetime.now().strftime("%Y-%m-%d"),
            "source": url,
            "scrapedAt": datetime.now().isoformat(),
            "visaTypes": visa_data
        }
    
    def save_country(self, data: dict):
        """Save country data to JSON file."""
        path = self.output_dir / f"{data['countryCode']}.json"
        with open(path, 'w') as f:
            json.dump(data, f, indent=2)
        print(f"Saved: {path}")
    
    def load_state(self) -> dict:
        """Load scraper state for resume."""
        if self.state_file.exists():
            return json.load(open(self.state_file))
        return {"completed": [], "failed": []}
    
    def save_state(self, state: dict):
        """Save scraper state."""
        with open(self.state_file, 'w') as f:
            json.dump(state, f, indent=2)
    
    def run(self):
        """Main scraper loop."""
        state = self.load_state()
        
        request_count = 0
        
        for code, name in COUNTRY_URL_MAP.items():
            # Skip if already done
            if code in state["completed"]:
                continue
            
            # Skip if no URL (e.g., US)
            if name is None:
                continue
            
            print(f"Scraping {code} ({name})...")
            
            data = self.scrape_country(code, name)
            
            if data:
                # Validate
                errors = validate_country_data(data)
                if errors:
                    print(f"Validation errors: {errors}")
                
                # Save
                self.save_country(data)
                state["completed"].append(code)
            else:
                state["failed"].append(code)
            
            # Save state
            self.save_state(state)
            
            # Batch delay
            request_count += 1
            if request_count % DELAY_CONFIG["batch_size"] == 0:
                print(f"Batch delay ({DELAY_CONFIG['batch_delay']}s)...")
                time.sleep(DELAY_CONFIG["batch_delay"])
        
        print(f"Done. Completed: {len(state['completed'])}, Failed: {len(state['failed'])}")

if __name__ == "__main__":
    scraper = VisaScraper()
    scraper.run()
```

---

## 11. TESTING PLAN

### 11.1 Unit Tests
- [ ] `normalize_visa_code()` handles all formats
- [ ] `parse_fee()` handles all fee formats
- [ ] `parse_validity()` handles months and years
- [ ] `parse_entries()` handles all entry formats
- [ ] `validate_country_data()` catches all error types

### 11.2 Integration Tests
- [ ] Fetch single country successfully
- [ ] Parse real HTML correctly
- [ ] Handle 404 gracefully
- [ ] Resume from saved state
- [ ] Batch delay works

### 11.3 Manual Validation
- [ ] Compare 5 countries against manual check
- [ ] Verify fee accuracy
- [ ] Verify validity accuracy

---

## 12. RISKS & MITIGATIONS

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| IP blocked | Medium | Use delays, rotate UA, respect limits |
| HTML structure changes | Low | Multiple selector fallbacks, alerts on parse failure |
| Missing countries | Medium | Log 404s, manual URL lookup |
| Incorrect parsing | Medium | Validation rules, manual spot checks |

---

## 13. EXECUTION PLAN

| Step | Action | Est. Time |
|------|--------|-----------|
| 1 | Build country URL mapping | 30 min |
| 2 | Implement scraper | 1 hour |
| 3 | Test on 5 countries | 15 min |
| 4 | Run full scrape (~200 countries) | 30-60 min |
| 5 | Validate results | 15 min |
| 6 | Fix failures manually | 15 min |

---

**Ready for QA Review ✅**

---

*Scraper Design by Cookie 🍪 | 2026-02-17*
