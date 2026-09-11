# SmartRecruiters Posting API

Endpoint: `https://api.smartrecruiters.com/v1/companies/<companyId>/postings` (no key).

Measured on **2026-09-11**.

```
curl 'https://api.smartrecruiters.com/v1/companies/BoschGroup/postings?limit=100&offset=0&country=de'
```

## 1. `limit` is capped at 100 without a word

- `limit=101`, `200` and `1000` all return 100 rows. `offset` does reach `totalFound`: BoschGroup's 4,839 postings came back in 49 pages, in 17 seconds, as 4,839 distinct ids.
- Company ids are case-insensitive (`ubisoft2` = `Ubisoft2`); the canonical spelling is in `company.identifier`.

## 2. A company that does not exist returns 200 and 0

- **Sent:** made-up and wrong company ids.
- **Got:** `{"totalFound":0,"content":[]}` with HTTP 200, exactly like a real company with no open roles.
- **Why it hurts:** a typo reads as "not hiring".
- **Avoid:** check existence separately. `https://careers.smartrecruiters.com/<ID>/api/more?page=1` returns 200 for a real company, even one with 0 postings (body empty), and 404 for an unknown id. We checked this against 50 companies.

## 3. Filters: some honest, some case-sensitive, some ignored

Checked against full downloads counted locally:

| Sent | Got | Correct |
|---|---|---|
| `country=de` | 780 | 780 |
| `country=DE` | **0** | 780 |
| two countries (`de&fr` or `de,fr`) | **0** | |
| `city=Stuttgart` | 57 | 57 |
| `city=stuttgart` | **0** | 57 |
| `location=`, `releasedDate=`, unknown parameters | **ignored**, full list returned | |
| `department=<word>` | 400 (numeric ids only) | |

- **Avoid:** lowercase country codes and query one country at a time. Filter city, location, date and department locally.

## 4. `q` is full text with OR semantics

- `q=montreal` returned 151 postings, and none of them had Montreal in the title.
- Two words mean either one: `game` 137, `designer` 137, `game designer` 179. Quoting the phrase returned 29.

## 5. `releasedDate` is the first-published date

- No two postings shared the same millisecond. The oldest quarter of posting ids (1,209) had no dates in the last 30 days. List and detail timestamps agreed 40 of 40. So it is safe to use for "new in the last N days".

## 6. One job, several postings

- One `refNumber` can carry several postings, in different languages (`fr-CA` and `en`) or at different sites. Ubisoft2 had 96 such groups in 291 postings; BoschGroup had 100 groups in 4,839, the largest with 10 postings.
- Grouping by `refNumber` alone also merges differently titled jobs. For DeliveryHero over 7 days, grouping by `refNumber` + title gave 135, the correct count; `refNumber` alone gave 127.

## 7. Posting details

- `/postings/<id>` has the full description, in 4 HTML sections, at about 0.3 to 0.45 seconds per call. 80 detail calls at 8-way parallelism took 3.6 seconds, with no 429.
- An unknown posting returns 404; a posting id requested under the wrong company returns 400.
- Postings added while you page shift the offsets, and one posting can be skipped. If the count comes up short, re-read.

## 8. Personal names in the payload

- `creator.name` is the recruiter's personal name.
- `customFields` held a named individual on one large employer's board, and the value sat in a select-type field (one carrying a `valueId`), so keeping only select-type fields does not remove names.
- **Avoid:** drop both unless you have a reason to keep them.
