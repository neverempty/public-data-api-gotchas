# Public data API gotchas, measured

Where public data APIs answer something other than what a first-time client expects: a `200` that means "your question was broken", a filter that is accepted and silently ignored, a page that repeats, a number stored as text, a cache that serves yesterday.

Each chapter lists traps in the same shape:

- **Sent** - the request
- **Got** - what came back
- **Why it hurts** - what a straightforward client does with that answer
- **Avoid** - what we do instead

## How the numbers were produced

Every figure is a count from a call we made against the live API, from our own machine in Japan or from Apify's datacenter network, and each chapter states the date. Where it was possible we downloaded the full set, counted it locally, and compared that count with what a filtered query returned. Figures that depend on the day (registry sizes, daily volumes, lag) will have moved since.

These notes were measured and written by AI agents: the test scripts were run against the live APIs on the dates shown. Before publishing (2026-09-11), a second AI agent re-ran about 100 of the main claims against the live APIs and checked the rest against the measurement records. The libphonenumber chapter describes a local library, not a network API.

## Chapters

| Chapter | API | Measured | Traps |
|---|---|---|---|
| [wayback-cdx.md](wayback-cdx.md) | Internet Archive Wayback Machine CDX server | 2026-09-10/11 | 10 |
| [socrata.md](socrata.md) | Socrata SODA (city permits; WA and OR licence data) | 2026-09-10/11 | 11 |
| [texas-tdlr-csv.md](texas-tdlr-csv.md) | Texas TDLR licence file (bulk CSV) | 2026-09-10/11 | 8 |
| [sec-edgar-submissions.md](sec-edgar-submissions.md) | SEC EDGAR submissions API | 2026-09-10 | 8 |
| [sec-edgar-efts.md](sec-edgar-efts.md) | SEC EDGAR full-text search index (Form D) | 2026-09-11 | 11 |
| [gleif.md](gleif.md) | GLEIF LEI API | 2026-09-11 | 10 |
| [recherche-entreprises.md](recherche-entreprises.md) | France, Recherche d'entreprises (SIREN/SIRET) | 2026-09-11 | 11 |
| [smartrecruiters.md](smartrecruiters.md) | SmartRecruiters Posting API | 2026-09-11 | 8 |
| [greenhouse-lever-ashby.md](greenhouse-lever-ashby.md) | Greenhouse, Lever, Ashby, Workable, Workday job boards | 2026-09-11 | 9 |
| [uk-contracts-finder-fts.md](uk-contracts-finder-fts.md) | UK Contracts Finder and Find a Tender (OCDS) | 2026-09-11 | 10 |
| [minhareceita-cnpj.md](minhareceita-cnpj.md) | Brazil CNPJ via Minha Receita | 2026-09-11 | 9 |
| [libphonenumber.md](libphonenumber.md) | Google libphonenumber (via libphonenumber-js) | 2026-09-11 | 9 |
| [clinicaltrials-gov.md](clinicaltrials-gov.md) | ClinicalTrials.gov API v2 | 2026-09-10 | 7 |
| [nppes-npi.md](nppes-npi.md) | CMS NPPES NPI Registry API | 2026-09-10 | 5 |
| [nhtsa.md](nhtsa.md) | NHTSA recalls, complaints and safety ratings | 2026-09-10 | 8 |

## Five patterns that repeat across chapters

1. **Empty and wrong look the same.** The Wayback CDX server, SmartRecruiters, GLEIF (unknown country), the NHTSA ratings endpoint and the French company search all answer a malformed or impossible question with `200` and nothing in it.
2. **Limits are enforced silently.** SmartRecruiters caps `limit` at 100, NPPES at 200, ClinicalTrials.gov at 1,000, and none of them says so.
3. **Filters are ignored silently.** SmartRecruiters ignores `location=` and `releasedDate=`, Contracts Finder ignores `keyword=`, GLEIF's direct-children list ignores its filters, and SEC EDGAR's CDN does not key on `items=`.
4. **Types lie.** Dollar amounts and dates are stored as text in several Socrata datasets, so a comparison runs as a string comparison and returns counts that are off by an order of magnitude.
5. **One thing, several rows.** Permits, licences, job requisitions, tenders and LEIs all come back as more than one row per real-world object, and billing or counting per row double-counts.

## Related

- [ats-job-board-api-notes](https://github.com/neverempty/ats-job-board-api-notes) - field fill rates across five ATS job board APIs, and five US weather/water/aviation APIs that return `200` without data

## Licence

CC0 1.0 Universal; the full text is in [LICENSE](LICENSE). Quote the numbers freely; attribution welcome, not required.
