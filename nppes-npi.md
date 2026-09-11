# CMS NPPES NPI Registry API

Endpoint: `https://npiregistry.cms.hhs.gov/api/` (no key; US government, public domain).

Measured on **2026-09-10**.

## 1. Errors come back as HTTP 200

- **Got:** HTTP 200 with `{"Errors":[{"description":"..."}]}` and no `result_count` field at all. Rejections we saw:

| Sent | Description |
|---|---|
| no criteria | `No valid search criteria provided` |
| `state=CA` only | `Field state requires additional search criteria` |
| unknown taxonomy | `No taxonomy codes found with entered description` |

- **Why it hurts:** a client that checks the status and then reads `results` tells the user "no providers found" for a search that was refused.
- **Avoid:** check for `Errors` before anything else.

## 2. A real zero looks different

- A search that really matches nothing returns `{"result_count":0,"results":[]}` with no `Errors`, so the two cases can be told apart.

## 3. `limit` is capped at 200 silently

- `limit=201` and `limit=1200` both return 200 records with no error.
- **Avoid:** page in steps of 200 with `skip`, which still worked at 2,000.

## 4. State alone is not a search

- Add a city, postal code, last name, organisation name or taxonomy.

## 5. Individuals and organisations have different shapes, and yes means `"YES"`

- `basic` holds name and credentials for individuals (NPI-1) but legal name and authorised official for organisations (NPI-2).
- Sole proprietor is `"YES"` / `"NO"`, not `"Y"` / `"N"`. Checking for `Y` gave `null` on every row; the real split was YES 37, NO 47, not stated 116.

---

Actor that handles these traps: https://apify.com/neverempty/npi-registry-scraper
