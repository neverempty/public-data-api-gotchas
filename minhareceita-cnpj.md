# Brazil CNPJ via Minha Receita

Endpoint: `https://minhareceita.org/<cnpj>`, a public API built on the Receita Federal open CNPJ data. No key. It is a volunteer-run instance with no service guarantee.

Measured on **2026-09-11**.

```
curl https://minhareceita.org/33000167000101
curl https://minhareceita.org/updated     # {"message":"2026-08"}
```

## 1. 400 and 404 mean different things

- A malformed number or wrong check digits returns **400** `{"message":"CNPJ 33.000.167/0001-02 inválido."}`.
- Valid check digits with no such registration return **404** `... não encontrado.`.
- **Avoid:** validate check digits yourself before sending, and report only a 404 as "not registered".

## 2. CNPJs are alphanumeric from July 2026

- Since July 2026 (IN RFB 2.229/2024), the first 12 characters can include uppercase letters. Check digits are computed from each character's ASCII code minus 48.
- Receita's own example `12.ABC.345/01DE-35` returns 404 (accepted as well-formed); changing the last digit to `36` returns 400. Lowercase is accepted.
- **Why it hurts:** a digits-only validator tells users that genuine new numbers are malformed.

## 3. Formatting does not matter

- `33.000.167/0001-01` and `33000167000101` return the same 200.

## 4. There is no email

- `email` was null on 440 of 440 records.
- `ddd_telefone_1` was filled on 290 of 440, sometimes with meaningless values like `"00"`.

## 5. Sole proprietors are people

- For legal nature 2135 (Empresário Individual, including MEI), the company name is the owner's name. In 55 of 440 records the name also contained the owner's unmasked 11-digit CPF.
- Legal nature 4xxx is also natural persons, e.g. 4090 for election candidates.
- **Avoid:** decide what to publish from these records under LGPD before storing them.

## 6. Shareholders

- `qsa[].identificador_de_socio`: 1 = legal entity (including foreign, with a country), 2 = natural person, 3 = foreign natural person.
- Natural-person shareholders come with a masked CPF, an age band and a legal representative. One record named the mother of a minor shareholder.

## 7. The activity code loses its leading zero

- `cnae_fiscal` is numeric, so leading zeros are dropped: 4 of 450 were 6 digits (`0600001` became `600001`).
- **Avoid:** pad to 7 digits.

## 8. Zero capital means "not declared"

- `capital_social` was 0 on 57 of 450, including 37 limited companies (2062), which are legally required to have capital.
- **Avoid:** map 0 to `null`.

## 9. Speed and alternatives

- 60 sequential requests with no pause all returned 200, about 270 ms each. In our production runs one lookup took about 0.62 seconds end to end. We saw no 429.
- `/updated` returns the month of the underlying Receita release; stamp it on every row.
- The alternative `publica.cnpj.ws` allows 3 per minute (`x-ratelimit-limit: 3`; the 4th request gets 429 with `retry-after: 60`). BrasilAPI's robots.txt disallows `/api/*`.
- **Avoid:** send one request at a time with a pause; the service is free and has no guarantee.
