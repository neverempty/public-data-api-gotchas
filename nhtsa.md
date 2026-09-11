# NHTSA: recalls, complaints and safety ratings

Three endpoints on the NHTSA public API (no key; US government, public domain).

Measured on **2026-09-10**.

## 1. The same API writes dates in opposite orders

We checked 1,873 date values:

| Endpoint | Field | Format | Evidence |
|---|---|---|---|
| Recalls | `ReportReceivedDate` | **DD/MM/YYYY** | first part > 12 in 30 of 39; second part > 12 in 0 |
| Complaints | `dateOfIncident`, `dateComplaintFiled` | **MM/DD/YYYY** | second part > 12 in 1,038 of 1,834; first part > 12 in 0 |

- **Why it hurts:** one parser for both shifts dates by up to 11 months.
- **Avoid:** give each field an explicit format and never guess.

## 2. Model names differ between endpoints

2021 Ford:

| `model=` | Recalls | Complaints | Safety ratings |
|---|---|---|---|
| `F-150` | 29 | 400 (unknown model) | 200, `Count` 0 |
| `F-150 SUPER CREW` | 400 (unknown model) | 1,001 | 2 |

Re-checked on 2026-09-11.

- Some vehicles have no spelling that works on all three endpoints.
- **Avoid:** when one endpoint returns nothing, check whether another endpoint accepts the same model before saying "no records".

## 3. Safety ratings never return 400

- `/SafetyRatings/modelyear/2022/make/Zorblax/model/Nope` returns 200 with `Count 0`. On this endpoint alone, "not rated" and "no such vehicle" look the same.
- **Avoid:** send the same vehicle to the recalls endpoint. A 400 there means the input is wrong; a 200 means the vehicle exists and is unrated.

## 4. Recalls and complaints reject unknown vehicles

- An unknown make, model or year returns 400 there. Keep that separate from fetch failures, and do not report it as "no recalls".

## 5. Complaint VINs are 11 characters

- All 221 complaint VINs we checked had 11 characters. A real VIN has 17; the vehicle-specific tail is removed.
- **Avoid:** call the field a VIN prefix, not a VIN.

## 6. The model list has duplicates

- `/products/vehicle/models?modelYear=2024&make=ford&issueType=r` returned 70 entries on 2026-09-11, with 29 names listed more than once.
- **Avoid:** de-duplicate model names before using them as a list of distinct models.

## 7. The count field changes case between endpoints

- Recalls and safety ratings return `Count`. Complaints and the model list return lowercase `count` (on 2026-09-11, 1,001 alongside 1,001 complaint rows). Recalls put rows in `results`, safety ratings in `Results`.
- **Avoid:** read both spellings, or count the array.

## 8. Ratings take two steps; nothing is paged

- Safety ratings return a list of `VehicleId`s for a year, make and model, and each id needs its own call.
- None of the three endpoints pages: one response holds everything (recalls `Count=29` with 29 results).

---

Actor that handles these traps: https://apify.com/neverempty/nhtsa-vehicle-recalls
