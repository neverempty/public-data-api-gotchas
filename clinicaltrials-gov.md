# ClinicalTrials.gov API v2

Endpoint: `https://clinicaltrials.gov/api/v2/studies` (no key; US government, public domain).

Measured on **2026-09-10**. The registry then held 602,104 studies, of which `cancer` matched 123,154.

## 1. About 40% of dates have no day

- Of 200 cancer studies, 117 had a full date, 81 had only year and month (`2011-12`), and 2 had none.
- **Why it hurts:** parsing `2011-12` as `2011-12-01` invents a day the registry does not have.
- **Avoid:** keep the source string and add a precision field (`day`, `month`, `unknown`).

## 2. An empty search returns the whole registry

- With every filter blank, the API returns all 602,104 studies. A variable that failed to set becomes a full export.
- **Avoid:** refuse to send a search with no filter.

## 3. A comma between values of one facet keeps only the last

Measured on `cancer`:

| Sent | Studies |
|---|---|
| `phase:2` | 39,750 |
| `phase:3` | 10,989 |
| `phase:2,phase:3` | **10,989**, phase 2 gone |
| `phase:2,phase:3,phase:4` | **2,597**, only phase 4 |
| `phase:2 3 4` | **51,862** |

- **Avoid:** join values of one facet with spaces (OR) and join different facets with commas (AND).

## 4. An unknown term returns 0, not an error

- `query.cond=zzznotadisease` returns HTTP 200 with `totalCount: 0`. A typo looks exactly like a disease nobody studies.

## 5. `pageSize` is capped at 1,000 silently

- Asking for 2,000 returns 1,000 with no error. Count what came back, not what you asked for.

## 6. Invalid filter values are rejected honestly

- They return HTTP 400 with `Invalid value in parameter ...` in the body. Do not retry a 400.

## 7. Paging and shape

- Paging uses `nextPageToken`; page 2 had 0 overlaps with page 1.
- Each study is 12 nested modules. Studies without a phase (every observational study) should get an empty list, not an invented `N/A`.

Filters narrow rather than being ignored: `RECRUITING` 18,755, a Japan location 3,105, phase 3 10,989 (each applied on its own to `cancer`).

---

Actor that handles these traps: https://apify.com/neverempty/clinical-trials-scraper
