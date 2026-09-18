# HTRC Extracted Features — stubbytree HTTPS + POS aggregate

Companion to `SKILL.md` §5. Use this file when the harness can download,
decompress, and aggregate **off-context**. Do **not** invent pairtree URLs or
guess EF hosts.

## Optional metadata check (not enough for themes)

```
GET https://data.analytics.hathitrust.org/extracted-features/20250321/{htid}
Accept: application/json
```

Returns small JSON (`title`, `accessRights`, `numPages`, …). Confirms the volume
is in EF 2.5; does **not** include token counts.

## Download full EF (stubbytree HTTPS)

1. Clean the HTID for the path: replace `:` → `+`, `/` → `=` (usually a no-op).
2. Split on the first `.`: `prefix` = library code (e.g. `uc1`), `rest` = remainder.
3. `chars` = every 3rd character of `rest`, starting at index 0
   (`rest[0]`, `rest[3]`, `rest[6]`, …). Example:
   `nyp.33433070251792` → `chars=33759` →
   `nyp/33759/nyp.33433070251792.json.bz2`.
4. `GET https://data.analytics.hathitrust.org/features-2025.04/{prefix}/{chars}/{cleaned}.json.bz2`
   (binary bzip2). Example Kuhn volume:
   `…/features-2025.04/uc1/32350/uc1.31822031154305.json.bz2`.
5. Decompress bz2 → JSON-LD. Schema: `metadata` (title, pubDate, …),
   `features.pageCount`, `features.pages[]`. Each page has
   `body.tokenPosCount`: `{ "<token>": { "<POS>": count, ... }, ... }`.

**Do not** use rsync-only paths, pairtree layouts, or invented `/features/{htid}`
URLs — those 404.

## Aggregate fingerprint

Over **body** `tokenPosCount` only (ignore header/footer):

- Lowercase tokens; keep alphabetic tokens with length ≥ 3.
- **Themes (`topThemes`, top 20):** POS in `{NN, NNS}`; drop this English stop
  list: `the and a an to of it is was were be been being have has had do does did
  in on at by for with from up out down into over under again about after before
  as this that these those there here i you he she they we me him her them us my
  your his its our their not no nor so than then too very can will would could
  should may might must just only also said one two three who whom which what
  when where why how all any both each few more most other some such own same`
- **Names (`topNames`, top 15):** POS in `{NNP, NNPS}`; do **not** apply the stop
  list.
- Sum counts across pages; sort descending.

Also set `title` / year from `metadata` (or metadata URL), `pageCount` from
`features.pageCount`, `readUrl` =
`https://babel.hathitrust.org/cgi/pt?id={htid}`.
