# France: Recherche d'entreprises (SIREN / SIRET)

Endpoint: `https://recherche-entreprises.api.gouv.fr/search`, run by DINUM on INSEE Sirene and RNE open data (Licence Ouverte 2.0). No key.

Measured on **2026-09-11**.

```
curl 'https://recherche-entreprises.api.gouv.fr/search?q=boulangerie&departement=69&per_page=25&page=1'
```

## 1. 25 per page, 10,000 per search

- `per_page` accepts 1 to 25 (26 returns 400), and `page * per_page` may not exceed 10,000 (page 401 returns 400).
- `total_results` is capped too: `departement=75` reports exactly 10000.
- **Avoid:** split large searches by department, then by NAF section.

## 2. Splitting by department loses companies

- Department filters match establishments in France, so a company whose head office and all establishments are abroad is not reached by any department.
- Measured on legal form 3120 (foreign companies registered in France): the NAF sections add up to 34,586 companies, but a department split returned 31,702 rows and gave no sign that anything was missing.
- For comparison, `q=airbus` finds 245 companies, and 35 of them have their head office abroad (`siege.code_pays_etranger` filled, no `departement`; re-checked 2026-09-11).
- **Avoid:** after the department pass, run a second pass by section alone. Compare each slice with its reported total and flag any remaining gap.

## 3. `total_results` can overstate

- `code_postal=76450` reports 4,104, but page 164 has 16 rows and page 165 has 0, for 4,091 in all.
- **Avoid:** stop on a short or empty page, and do not call that shortfall "incomplete".

## 4. Order is stable

- The same search read twice returned the same 495 rows in the same order, so page numbers are safe to use.

## 5. `q` matches whole words, and more than names

- `renau` returns 239 hits and none of them is RENAULT; `renaullt` returns 0, so there is no fuzzy matching.
- Case, accents, hyphens and quotes are ignored: `société générale` = `SOCIETE GENERALE` = `societe-generale` = 8,507. A `q` under 3 characters returns 400.
- `q` also matches addresses, officers and elected officials. `q=renault` in 76450 returned 9 companies, including one whose officer is named Renault and a commune.
- **Avoid:** if the user asked for names, filter the name fields locally.

## 6. Bad values error honestly; bad parameter names do not

- Unknown values of known parameters (department 999, NAF 99.99Z...) return 400 with the list of valid values.
- Unknown parameter **names** are ignored silently: `foo=bar` returns the same 10,000. Unknown commune codes return 200 and 0 (`69999`).
- Paris, Lyon and Marseille return far fewer companies than they hold, because the source files them under their arrondissements. On 2026-09-11, `code_commune=75056` returned 9, `69123` returned 1 and `13055` returned 11, while the single arrondissement `75111` alone hit the 10,000 cap. With `q=boulangerie`, `75111` returned 57 and `75056` returned 0.
- **Avoid:** expand the three city codes into arrondissements, and check parameter names against the documentation.

## 7. A SIREN search returns the company without its establishments

- Searching for Renault SAS's SIREN returns an empty `matching_etablissements` list, even with `limite_matching_etablissements=100` (the company has 224 establishments).
- With a location filter, it lists the establishments inside the area, 100 at most (Renault SAS in department 92: 20).
- **Avoid:** take counts from `nombre_etablissements`, and never present `matching_etablissements` as "all establishments".

## 8. Location filters pick establishments, not head offices

- `departement=75,92` with `boulangerie` returns BOULANGERIES PAUL, whose head office is in department 59.

## 9. Non-public records use a placeholder string

- Fully non-public companies are absent. Partly public ones carry `[NON-DIFFUSIBLE]` in the address, acronym or coordinates.
- **Avoid:** convert the placeholder to `null` and flag it.

## 10. One company's SIRETs fail the Luhn check

- All 4,091 SIRENs in 76450 passed Luhn, and in a separate check 8,759 real SIRETs raised no false errors. The only failure was La Poste (SIREN 356000000), where INSEE's rule is "sum of the 14 digits divisible by 5".
- A valid-Luhn number can exist with an empty head office and no status (`123456782`).

## 11. The rate limit hits cloud IPs quickly

- The documented limit is 7 requests per second per IP and 30 per second per ASN. From a cloud datacenter, 3 quick requests already drew a 429 with `retry-after: 4`.
- **Avoid:** send one request at a time with a short pause, and honour `retry-after`.

## Personal data

Officers who are natural persons come with birth year and month (`YYYY-MM`) and nationality. Decide what you actually need before storing it.
