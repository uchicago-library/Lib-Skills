---
name: catalog-search
description: >-
  Search the University of Chicago Library catalog (VuFind) and enrich beyond
  the record: WikiData author context; Internet Archive full-text badges;
  verify-first Project Gutenberg / IA discovery; PubMed topic evidence for
  biomedical/clinical questions (recent reviews, MeSH, sample papers alongside
  library holdings); HathiTrust/HTRC content fingerprints for in-copyright
  works. Use when the user asks about library holdings, "does the library have
  X by Y", books on a topic, narrowing by format/year/language, author context,
  public-domain full text, "what does this book cover" (HathiTrust search-only),
  or biomedical / health evidence such as "I'm reviewing type 2 diabetes
  treatments — what's the recent evidence?", CRISPR, Alzheimer disease, or
  other clinical topics (catalog first, then PubMed). Default catalog:
  https://catalog.lib.uchicago.edu/vufind (HTTP steps; no Python/venv).
---

# catalog-search (Lib-Skills)

Search the UChicago Library catalog and **enrich** beyond the record:

- **WikiData** (inline, top-N): author one-liner, notable works, VIAF.
- **Full text** (on demand, public-domain): IA badge; pull to read / summarize / Q&A.

Covers and basic availability are baseline catalog jobs — enrichment goes
**beyond** the record.

**Execution:** You perform HTTP and normalize into the schema below. **No Python /
scripts / venv.** Follow badges, `searchUrl` / `readUrl`, honesty, and envelope
rules exactly. Higher token use is fine.

## When to use

- Catalog search / holdings / topic browse / title+author asks; filter by format,
  language, year.
- Author context ("who is #2?", "what else did they write?").
- Full-text pull / summarize; verify-first findtext when badge missed (including
  specific-title holdings asks — users rarely say “I’d like to read it”).
- "What does this book cover?" when no public full text → HTRC EF via HTTP.
- Biomedical / clinical "recent evidence" → catalog first, then PubMed
  `topicEvidence` (do not skip to PubMed-only).

Do **not** use for live FOLIO availability, or to claim full text without the badge.

## Config (baked defaults)

| Key | Default | Notes |
|---|---|---|
| `catalog_base` | `https://catalog.lib.uchicago.edu/vufind` | UChicago default |
| `probe_depth_n` | `5` | Top-N results to annotate eagerly |
| `http_timeout` | ~20s | Soft; fail-soft on timeout |
| `fulltext_timeout` | ~90s | For large OCR/plaintext pulls |

**Origin** for permalinks: scheme + host of `catalog_base` (e.g.
`https://catalog.lib.uchicago.edu`). Permalink = `origin` + `recordPage` — *not*
base + `recordPage` (`recordPage` already includes `/vufind`).

**User-Agent:** descriptive string; optional contact mailto if known. Never invent
catalog hits if the API is unreachable (403 / challenge HTML / network).

**Inline enrichers (eager):** `wikidata`, `openlibrary_ia`.
**Opt-in / on-demand:** HathiTrust badge (htrc intent), PubMed (pubmed intent),
findtext discovery, fulltext pull, HTRC analyze (when EF obtainable).

---

## Workflow

### 1 & 2 — Search + filter (HTTP)

**Agent should:** `GET {catalog_base}/api/v1/search` with query params:

| Param | Value |
|---|---|
| `lookfor` | user query string |
| `type` | `AllFields` \| `Title` \| `Author` \| `Subject` \| `ISN` |
| `limit` | e.g. `10` |
| `page` | `1` (default) |
| `field[]` | repeat for each needed field (see list) |
| `filter[]` | optional, repeatable VuFind filters |
| `facet[]` | optional: `format`, `language`, `publishDate` |
| `sort` | optional |

**Required `field[]` values** (identifiers are absent from VuFind defaults):

```
id, title, authors, formats, languages, subjects, publicationDates,
callNumbers, recordPage, isbns, cleanIsbn, oclc, cleanOclcNumber
```

Map natural language → `type` and `filter[]`:

| User says | `type` | `filter[]` |
|---|---|---|
| "by Jane Austen", "author …" | `Author` | |
| "the book titled …" | `Title` | |
| "about / on \<topic\>" | `Subject` (or `AllFields`) | |
| an ISBN/ISSN | `ISN` | |
| "just books" | | `format:"Book"` |
| "print copies only" | | `format:"Print"` |
| "online / e-book" | | `format:"E-Resource"` |
| "in English" | | `language:English` |
| "since 2015" | | `publishDate:[2015 TO *]` |
| "between 2015 and 2020" | | `publishDate:[2015 TO 2020]` |

Filter values use VuFind syntax (quote multi-word values: `format:"Book"`).
Request facets to get counts so you can offer narrowing: *"9,911 are books,
5,945 e-resources — want to limit?"*

**Always annotate** for user-facing searches (WikiData + full-text badge on top
`N`). Skip annotation only for a quick count or internal lookup.

#### Normalize VuFind → record schema

Match this schema. From each raw record:

- `id` ← `id`
- `title` ← `title`
- `authors[]` ← flatten `authors.primary|secondary|corporate`: if a bucket is a
  dict, take its **keys** as names; if a list/string, take values. Order:
  primary → secondary → corporate. Deduplicate preserving order.
- `year` ← first 4-digit year found in `publicationDates[]` (regex `(\d{4})`)
- `format` ← first of `formats[]`; keep full `formats[]`
- `language` ← first of `languages[]`
- `subjects[]` ← for each entry: if list of parts, join with ` -- `; else string;
  cap at **6**
- `callNumber` ← first of `callNumbers[]`
- `permalink` ← `origin` + `recordPage`
- `identifiers` ← `{ isbn: isbns[], cleanIsbn, oclc: oclc[], cleanOclcNumber }`
- `annotations`: `{}` then fill in annotate stage
- `actions`: `[]` then fill when badges register actions

#### Build `searchUrl`

Human results page mirroring the same query:

```
{catalog_base}/Search/Results?lookfor=…&type=…&filter[]=…
```

(url-encode; include the same filters you sent to the API.)

#### Envelope after search

```
{ query, type, resultCount, searchUrl, records[], facets? }
```

Parse facets (if requested) as `{ field: [ {value, count}, ... ] }`.

If the response is not JSON (bot-check HTML, 403, empty), **stop**: tell the user
catalog access failed (VPN / campus network) and do not invent records.

---

### 3 — Annotate top-N (parallel, fail-soft)

For the first `probe_depth_n` (default **5**) records, run probes **in parallel**.
A failed/slow probe drops its annotation only — never break the search.

#### 3a. WikiData (always-on inline)

1. Take primary author `original = authors[0]`.
2. **`clean_author_name(original)`:**
   - Strip life dates: `,?\s*\d{3,4}\s*-\s*(?:\d{3,4})?\.?`
   - Strip parentheticals: `\([^)]*\)`
   - Strip trailing commas/whitespace
   - If a comma remains: invert `Last, First` → `First Last`
   - Collapse whitespace
   - Example: `Tolkien, J. R. R. (John Ronald Reuel), 1892-1973` → `J. R. R. Tolkien`
3. `GET https://www.wikidata.org/w/api.php` with:
   `action=wbsearchentities&search=<cleaned>&language=en&uselang=en&format=json&type=item&limit=5`
4. **Name-match guard** (`_name_matches`): only keep hits whose label matches:
   - Significant tokens = lowercase word tokens length ≥ 3, not pure digits
   - If `original` contains a comma (personal name): surname = last token of
     cleaned; **surname must appear in label tokens**; if any given-name tokens
     exist, **at least one** must appear in label tokens
   - Else (corporate/mononym): require non-empty significant-token overlap
   - Wrong-person → **suppress** (no annotation). Prefer missing over wrong.
5. Among surviving hits, prefer a description containing a **person word**:
   author, writer, poet, novelist, historian, scholar, philosopher, professor,
   journalist, philologist, playwright, scientist, academic, editor, translator,
   essayist, critic, sociologist, economist, linguist. Else take first surviving hit.
6. `GET` same API `action=wbgetentities&ids=<qid>&props=claims&format=json`
   - Notable works `P800` (≤5 unique QIDs), occupations `P106` (≤3), VIAF `P214`
     (first value)
7. Resolve claim QIDs to English labels:
   `wbgetentities&ids=<qid|…>&props=labels&languages=en&format=json`
8. Attach `annotations.wikidata`:
   `{ qid, label, summary, notableWorks?, occupations?, viaf?, url }`
   where `url` = `https://www.wikidata.org/wiki/{qid}` and `summary` = search
   description.
9. **Surface gate:** if `original` contains a comma (personal), require
   `summary` **or** `notableWorks` before attaching (occupations alone are not
   enough). For corporate/mononym, any of summary / works / occupations suffices.

**Present as:** *"Jane Austen — English novelist (1775–1817); also wrote Emma,
Persuasion, Sense and Sensibility."* (Use the actual `summary` / `notableWorks`
from the annotation.)

#### 3b. OpenLibrary → IA full-text badge (always-on inline)

**High precision only:** badge iff OpenLibrary reports `ebook_access == "public"`
with an `ia` list. Never badge on `borrowable` / `lendable` / restricted.

1. **`_clean_title(title)`:** trim at first `/` or `:` (subtitle / statement of
   responsibility noise).
2. **`lc_author(authors[0])`:** if comma-inverted, flip to `First Last`; drop
   parenthetical and digit runs from the first-name part.
3. `GET https://openlibrary.org/search.json` with
   `title=<cleaned>&author=<lc_author>&fields=key,title,ia,ebook_access,public_scan_b&limit=5`
4. If miss, retry with `isbn=<cleanIsbn or first isbn>` and same `fields`/`limit`.
5. **`_first_public_scan(docs)`:** first doc with `ebook_access == "public"` and
   non-empty `ia`; prefer docs that still have at least one **real scan** after
   filtering (see below); else fall back to a public doc with only placeholder ids.
6. **Scan filters / ranking** on `ia` ids:
   - Drop prefixes in `bwb_` (Better World Books listings — no OCR)
   - Rank score: `+10` if id matches library-scan pattern `[a-z]\d{4}[a-z]`;
     `-50` if id contains any of `synapseml`, `librivox`, `_dataset`,
     `spectrogram`, `audio`; minus `id.count("_")`; `bwb_` → `-100`
   - Pick highest-ranked id as `ocaid`; keep top 5 as `iaCandidates`
7. Attach:

```
annotations.openlibrary_ia: {
  fullText: true, access: "public", source: "Internet Archive",
  ocaid, readUrl: "https://archive.org/details/{ocaid}",
  iaCandidates[], olKey, matchedTitle
}
actions: [{ id:"fulltext", enricher:"openlibrary_ia",
  label:"Pull full text from the Internet Archive (read / summarize / Q&A)",
  params:{ ocaid } }]
```

**Badge copy (exact intent):** *"📖 Full text available — read it at `<readUrl>`,
or say the word and I'll pull it to read/summarize."*

#### 3c. Optional HathiTrust badge (user intent ≈ `--htrc`)

When the user asks for content-analysis availability, or you opt in for a demo:

1. Build id list from record (≤10, deduped): `oclc:{n}` for each `identifiers.oclc`,
   then `isbn:{n}` for each `identifiers.isbn`.
2. `GET https://catalog.hathitrust.org/api/volumes/brief/json/{oclc:…|isbn:…}`
   — pipe-join ids; URL-encode keeping `:` and `|` as safe.
3. Collect items with `htid`; `rights` = lowercased `rightsCode` or `rights`.
   **Full-view rights codes:** `pd`, `pdus`, `world`, `cc-by`, `cc-by-nd`,
   `cc-zero`, `cc-by-nc`, `cc-by-nc-nd`, `cc-by-sa`, `cc-by-nc-sa`.
   Sort full-view first; pick best.
4. Attach:

```
annotations.hathitrust: {
  contentAnalysis: true,
  source: "HathiTrust / HTRC Extracted Features",
  htid, rights, rightsNote,   # rightsNote ← usRightsString
  readUrl: "https://babel.hathitrust.org/cgi/pt?id={htid}",
  note: "full view" | "analyzable even though not readable (features only)"
}
actions: [{ id:"content_analysis", enricher:"hathitrust",
  label:"Analyze what this book is about (HTRC Extracted Features)",
  params:{ htid } }]
```

Honest miss rate ~7/8 for catalog→HT auto-join — absent badge is fine.

#### 3d. Optional PubMed topic evidence (biomedical only; ≈ `--pubmed`)

Set-level (top of envelope), not per-record.

**Heuristic:** disease, drug, organism, biological mechanism, clinical
intervention → yes; primarily cultural/historical/artistic → skip entirely
(don't even call, or call and self-gate).

1. `GET https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi`
   with `db=pubmed&term=<query>&retmode=json&retmax=20&tool=lib-skills&email=<optional>`
2. If `esearchresult.count < 50`, **omit** `topicEvidence` (self-gating floor).
3. Else search reviews last 5 years:
   term `({query}) AND review[pt]` with `mindate=<year-5>&maxdate=<year>&datetype=pdat`
4. `GET …/efetch.fcgi?db=pubmed&id=<pmids>&retmode=xml` → count MeSH
   `DescriptorName`s (top 8); take ≤3 sample reviews
   `{pmid, title, year, url: https://pubmed.ncbi.nlm.nih.gov/{pmid}/}`
5. Attach top-level:

```
topicEvidence: {
  source:"PubMed", topic, totalArticles, recentReviews,
  recentWindow: "<year-5>–<year>",
  topMeSH[], sampleReviews[{pmid,title,year,url}],
  searchUrl: "https://pubmed.ncbi.nlm.nih.gov/?term=<urlencode(query)>+AND+review%5Bpt%5D"
}
```

**Present catalog results first**, then PubMed as secondary context — never lead
with PubMed stats: *"The library has N items on this. PubMed also indexes ~…,
including ~… recent reviews; key themes: … — want the top reviews?"*

---

### 3 (present) — Present the results

Show essentials, then fold annotations as **enrichment, clearly attributed**:

- Core: `title` · `authors` · `year` · `format` · `permalink` · `callNumber`
- `annotations.wikidata` → one-liner bio + “also wrote”
- `annotations.openlibrary_ia` → full-text badge + `readUrl`
- `annotations.hathitrust` → content-analysis badge + `readUrl` (if opted in)

**Always give the human `readUrl`**, not just machine `source_url`.
**Always offer `searchUrl`** with counts, especially when truncated:
*"404 books by Michel Foucault — here are the top 10; see them all in the
catalog: \<searchUrl\>."*

Missing annotation = probe found nothing — **never invent**.

---

### 4 — Act: pull full text (on demand)

Only when the user asks, and preferably when the full-text badge fired (or after
confirmed findtext). Use `ocaid` from `actions[].params` / annotation.

**Internet Archive:**

1. `GET https://archive.org/metadata/{ocaid}` → `files[]`
2. Prefer file with `format == "DjVuTXT"`; else name ending `_djvu.txt`; else any
   `.txt` that is **not** a sidecar ending `_meta.txt`, `_files.txt`, `_reviews.txt`
3. `GET https://archive.org/download/{ocaid}/{filename}` (URL-encode filename) →
   full OCR text
4. If that scan lacks text, try next `iaCandidates` entry
5. Save to a working file if the harness allows; do **not** dump entire text to
   the user. Summarize / quote / answer from the text.
6. Report envelope:

```
{ source, id, title, chars, savedTo?, readUrl, source_url, head }
```

`readUrl` = `https://archive.org/details/{ocaid}`
`source_url` = raw text file URL (internal; do not present as the reading link)

**Re-resolve from catalog id:** `GET {catalog_base}/api/v1/record?id=<id>` with
the same `field[]` list, normalize, re-run OL probe, then pull.

**Project Gutenberg** (after findtext confirm):

1. `GET https://gutendex.com/books/{id}` → pick `text/plain` URL preferring
   `utf-8`, skip `.zip`
2. Fallback: `https://www.gutenberg.org/files/{id}/{id}-0.txt`
3. Strip boilerplate between
   `*** START OF (THE|THIS) PROJECT GUTENBERG … ***` and
   `*** END OF (THE|THIS) PROJECT GUTENBERG … ***` (case-insensitive)
4. `readUrl` = `https://www.gutenberg.org/ebooks/{id}`

---

### 4b — Find full text the badge missed (verify-first)

Discover free full-text **candidates** when the eager OL→IA badge did **not**
fire — **never auto-claim**. Prefer Gutenberg. Users rarely say “I’d like to
read it”; do **not** wait for that phrase.

**Run findtext (or at least offer / present candidates) when any of:**

1. The user asked about a **specific titled work** (title + author, or a clear
   unique title) — including plain holdings asks like “Does the library have
   *X* by *Y*?” — and the top hit(s) have **no** full-text badge.
2. The user explicitly wants full text / to read / summarize / quote a work
   with no badge (reprint-only catalog copies are the common case).
3. A search result is **clearly public-domain** (author life dates ending well
   before today, or an obviously historical work) with no badge — offer
   proactively: *“No IA badge on the catalog edition, but free text may be on
   Project Gutenberg — want me to check?”* (or just check and present
   candidates if the ask was already about that one work).

**Skip** findtext for broad subject/author browse (“books on X”, “everything by
Foucault”) unless the user picks a specific title afterward.

1. Resolve title + author from user or catalog record
   (`GET …/api/v1/record` if given a record id)
2. Helpers:
   - `surname_of`: last significant token of the part before comma (or last word)
   - `life_dates`: first `(\d{4})\s*-\s*(\d{4})?` in author heading
   - `title_tokens`: tokens length ≥ 4 excluding `{the,and,for,with,from,that,this}`
   - `author_matches`: same surname **and**, when both have given-name tokens
     (length ≥ 3, non-digit), at least one shared given-name token
   - `title_overlap_ok`: ≥2 shared title tokens, **or** (requested title has ≤1
     token and overlap has a token length ≥ 6)
3. **Gutenberg:** `GET https://gutendex.com/books?search=<surname + up to 3 title tokens>`
   - Require `author_matches`; require a plaintext format URL
   - Confidence:
     - **high** if (life-date birth matches **or** shared given name) **and**
       title overlap — notes accordingly
     - **medium** if title overlap only, or author confirmed but title differs
     - **low** if surname-only
4. **IA (library scans only):**
   `GET https://archive.org/advancedsearch.php` with
   `q=title:(…) AND creator:(surname) AND mediatype:texts&output=json&rows=10`
   and `fl[]=identifier,title,creator,year,collection,access-restricted-item`
   - Skip `access-restricted-item` true/`true`
   - Skip community collections: any collection equal to or prefixed by
     `opensource`, `folkscanomy`, `community`
   - Require `author_matches` + `title_overlap_ok`
   - Confidence at best **medium**; notes:
     `"library/institutional scan; creator + title match — confirm it's the right edition before relying on it"`
5. Rank: high → medium → low; at equal confidence, **Project Gutenberg before IA**
6. Present candidates with `matchNotes`; **confirm before pull**
7. After confirm → step 4 pull

Output shape:

```
{ query:{title, author}, candidates:[
  { source, id, title, author, authorDates|year, format, url, textUrl?,
    confidence: high|medium|low, matchNotes } ] }
```

---

### 5 — Act: HTRC content analysis (in-copyright OK; HTTP, no venv)

Complement to full text: theme + named-entity **fingerprint** from HTRC
Extracted Features (non-consumptive page-level token/POS counts — **not**
readable text). Use when the user asks what a book covers / is about and there
is no public full text (HathiTrust search-only, in-copyright, etc.).

**Honesty / fail-soft:** Prefer a harness **tool/sandbox** to download + bunzip +
aggregate **off-context**; only bring the tiny `topThemes` / `topNames` lists
into the reply. Follow `references/htrc-ef.md` for stubbytree HTTPS download +
POS aggregate; if the harness cannot bunzip/aggregate off-context, refuse
honestly — still offer `readUrl` (+ Bib-API rights / metadata). **Never invent
themes, names, pairtree URLs, or guess EF hosts.**

**Getting `htid`:**

1. From htrc-badge `params.htid`, or
2. From user HathiTrust URL (`pt?id=` query param, or `hdl.handle.net/2027/<htid>`), or
3. Catalog→Bib API join (same as badge) — honest miss ~7/8 → ask for HTID/URL

**Report shape** (vocabulary profile, not a summary of read text):

```
{ htid, title, year, pageCount, readUrl, rights?, topThemes[{term,count}], topNames[{term,count}] }
```

Example tone: *"Vocabulary centers on science, paradigm, theory; names include
Newton, Galileo. In-copyright — profile from word statistics only."*

---

## Notes & edge cases

- **Zero results** → no invented hits; offer broaden / drop filter / `AllFields`.
- **Author searches are broad** — use WikiData + permalinks to disambiguate.
- **`format` is multi-valued** (`Book` / `Print` / `E-Resource`).
- **Fail-soft** — one slow enricher never blocks search results.
- **VPN / 403 / Anubis** — catalog may need campus network; don't fake holdings.

---

## Output schema

Envelope: `{ query, type, resultCount, searchUrl, records[], facets?, topicEvidence? }`

Normalized record:

```
{ id, title, authors[], year, format, formats[], language, subjects[],
  callNumber, permalink,
  identifiers: { isbn[], cleanIsbn, oclc[], cleanOclcNumber },
  annotations: {
    wikidata?:       { qid, label, summary, notableWorks[], occupations[], viaf, url },
    openlibrary_ia?: { fullText:true, access:"public", source:"Internet Archive",
                       ocaid, readUrl, iaCandidates[], olKey, matchedTitle },
    hathitrust?:     { contentAnalysis:true, source?, htid, rights, rightsNote,
                       readUrl, note }
  },
  actions: [ { id:"fulltext", enricher:"openlibrary_ia", label, params:{ocaid} },
             { id:"content_analysis", enricher:"hathitrust", label, params:{htid} } ]
}
```

`topicEvidence?` (PubMed): `{ source:"PubMed", topic, totalArticles, recentReviews,
recentWindow, topMeSH[], sampleReviews[{pmid,title,year,url}], searchUrl }`

---

## Principles

- **Honesty.** Only state enrichment that is present. Full-text badge only for
  verified public-domain scans (`ebook_access == public`).
- **Fail-soft.** Missing annotation ≠ failed search.
- **Lean & on-demand.** Annotate top `N` only; never eager full-text across a set.
- **Harness-neutral.** Prefer parallel HTTP for probes; map to harness primitives.
- **Schema fidelity.** Badges, links, and the result envelope match this skill’s schema.

## Extending

New sources = document probe/act HTTP steps here with the same honesty bar.
