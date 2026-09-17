# catalog-search — Example Queries (Lib-Skills)

A tour of what the skill can do, for testing and demoing. Each example gives the
natural-language **ask**, what the **agent should** do (HTTP steps), what it
**shows**, and what to **look for**. Same scenarios as Lib-Bot EXAMPLES —
Python CLI invocations replaced by agent HTTP behavior.

## Intent quick reference

| User intent | Agent does |
|---|---|
| type | Map to VuFind `type`: AllFields · Title · Author · Subject · ISN |
| filter | Send `filter[]`: `format:"Book"`, `language:English`, `publishDate:[2015 TO *]` |
| facets | Request `facet[]=format,language,publishDate` → offer narrowing |
| annotate | Top-N WikiData + OL→IA full-text badge (default on for demos) |
| pubmed | Set-level PubMed `topicEvidence` (biomedical only; self-gating) |
| htrc | HathiTrust Bib-API “content analysis available” badge where joined |
| limit | Cap `limit` on search |

**Default `catalog_base`:** `https://dldc2.lib.uchicago.edu/vufind`

*(Catalog search needs VPN / campus reachability. Enrichment demos below that
only hit public APIs still prove badges without the catalog.)*

---

## 1. Search & disambiguation

**Ask:** "Find books by Tolkien."

**Agent should:** HTTP `GET {catalog_base}/api/v1/search` with
`lookfor=tolkien&type=Author&limit=5` + required `field[]`; annotate top-N
(WikiData + OL probe); build `searchUrl` for `/Search/Results`.

- **Shows:** catalog search → normalized records (title/author/year/format/
  permalink/identifiers) **+** WikiData inline enrichment, including
  disambiguation of people who share a surname.
- **Look for:** J. R. R. vs **Christopher** vs **Simon** Tolkien each get a
  *different, correct* one-line bio; real `permalink` catalog links; ISBN/OCLC
  under `identifiers`; top-level **`searchUrl`** to the live catalog UI.

**Ask:** "Anything by Toni Morrison / Borges / Foucault?" — swap the name; expect
bio + "also wrote" + VIAF when WikiData hits.

---

## 2. Filtering & facet narrowing

**Ask:** "Books about artificial intelligence, just books, published since 2015 —
and how could I narrow further?"

**Agent should:** HTTP search `lookfor=artificial intelligence&type=Subject` with
`filter[]=format:"Book"` and `filter[]=publishDate:[2015 TO *]`, plus
`facet[]=format,language,publishDate`, `limit=5`. Present facet counts and offer
next narrow (e.g. Print vs E-Resource).

- **Shows:** eager filters **and** facet counts for conversational narrowing.
- **Look for:** `resultCount` drops with filters; `facets` block with
  format/language/year counts.

---

## 3. Author context — WikiData (including honesty)

**Ask:** "Tell me about the authors in a Jane Austen search."

**Agent should:** HTTP search `Jane Austen novels`, `type=AllFields`, `limit=6`,
annotate; WikiData reconcile with `clean_author_name` + name-match guards +
person-word preference; suppress wrong-person (require summary/notableWorks for
personal names).

- **Shows:** genuine enrichment **and** honesty guard — prefer *nothing* over a
  wrong-person bio.
- **Look for:** Jane Austen (and notable critics like Robert Liddell) get correct
  bios; **minor/ambiguous authors get no WikiData block** (e.g. Marsh, Nicholas
  style mismatches suppressed).

**Public-only smoke (no catalog):** WikiData `wbsearchentities` for `Jane Austen`
should prefer Q36322 (“author, novelist, writer (1775–1817)”) over same-name
non-author hits.

---

## 4. Full text of a public-domain book — Internet Archive *(headline)*

**Ask:** "Find Pride and Prejudice and pull the full text — summarize chapter 1."

**Agent should:**

1. HTTP search `pride and prejudice`, `type=Title`, `limit=3`, annotate → expect
   `annotations.openlibrary_ia.fullText: true` + `fulltext` action / `ocaid` /
   `readUrl` (only if `ebook_access == public`).
2. On request: `GET archive.org/metadata/{ocaid}` → prefer DjVuTXT / `*_djvu.txt`
   → download OCR; summarize chapter 1 from the text; offer `readUrl`
   (`https://archive.org/details/…`). Optionally re-resolve via
   `GET {catalog_base}/api/v1/record?id=…` when a catalog id is known.

- **Shows:** two-tier model — cheap badge, expensive on-demand pull → read /
  summarize / Q&A from *real* text.
- **Look for:** badge + action in step 1; pull reports large `chars` (~700K+);
  never paste the whole book; always offer human `readUrl` (not raw
  `source_url`). Other classics: **Frankenstein, Dracula, Moby Dick, Walden, The
  Origin of Species.**

**Public-only smoke (no catalog):** OpenLibrary
`search.json?title=Pride and Prejudice&author=Jane Austen&fields=key,title,ia,ebook_access,public_scan_b`
should return a top work with `ebook_access=public` and many non-`bwb_` IA ids;
rank library-scan pattern `[a-z]\d{4}[a-z]` ahead of derivative ids.

---

## 4b. Find full text the badge missed — verify-first (Gutenberg / IA)

**Ask:** "Does the library have *An Essay Towards a Philosophy of Education* by
Charlotte Mason? I'd like to read it."

Catalog may only hold a reprint with **no** full-text badge. Discovery returns
**candidates to confirm**.

**Agent should:**

1. Catalog search for the title/author (optional); note missing badge.
2. HTTP discover: Gutendex `search` (surname + title tokens) + IA advancedsearch
   (library scans only; skip community collections); apply author_matches +
   title_overlap_ok; rank by confidence; present high Gutenberg and medium IA
   to confirm.
3. After user confirms: pull Gutenberg plaintext (strip PG boilerplate); offer
   `readUrl` on `gutenberg.org/ebooks/{id}`.

- **Shows:** high-precision badge can miss PD reprints; recall without lying.
- **Look for:** **`high`** Gutenberg candidate (#66369, Charlotte M. Mason
  1842–1923) + **`medium`** IA; honesty on *"School education"* by Charlotte
  Mason — reject false positives like unrelated “school” curricula (given-name +
  multi-token title guards).

---

## 5. Content analysis of a book you *can't* read — HathiTrust/HTRC *(headline)*

**Ask:** "Before I request Thomas Kuhn's *The Structure of Scientific
Revolutions*, what does it actually cover? The library only has it on HathiTrust
as search-only, so I can't read the text."

**Agent should:**

1. Obtain HTID from user URL
   `https://babel.hathitrust.org/cgi/pt?id=uc1.31822031154305` or
   `htid=uc1.31822031154305`.
2. Prefer: fetch HTRC Extracted Features (harness tooling /
   `htrc-feature-reader` if available) → compute `topThemes` / `topNames`.
3. If features **cannot** be obtained: say so honestly; still offer `readUrl` and
   Bib-API rights if known — **never invent themes**. This is the documented
   Lib-Skills gap vs Lib-Bot.

- **Shows:** complement to #4 — in-copyright fingerprint from non-consumptive
  stats when features are available; honest refusal when not.
- **Look for (when EF works):** themes like *science, **paradigm**, theory,
  research*; names *Newton, Lavoisier, Galileo, Einstein* — presented as a
  statistical profile. Alt HTIDs for PD demos: `njp.32101075725117`,
  `nyp.33433042068894`.

**The honest miss (catalog auto-join):**

**Agent should:** try Bib-API join from a record’s OCLC/ISBNs; when no volume
matches (~7/8), report clear failure and ask for HTID/URL.

**The badge in a search:**

**Ask:** annotate Peloponnesian War with HTRC intent.

**Agent should:** Title search + Bib-API probe on top-N; badge only where joined;
same action label as Lib-Bot.

---

## 6. Topical evidence — PubMed (with the honest negative)

**Ask:** "I'm reviewing type 2 diabetes treatments — what's the recent evidence?"

**Agent should:** Subject search `type 2 diabetes`, `limit=5`, plus PubMed
esearch/efetch; attach `topicEvidence` only if total ≥ **50**.

- **Shows:** set-level biomedical evidence — totals, recent reviews, MeSH,
  samples; catalog first, PubMed second.
- **Look for:** `topicEvidence` (~280k articles, ~18k recent reviews; MeSH like
  *Hypoglycemic Agents, GLP-1 Receptor Agonists*). Also try `"CRISPR gene
  editing"` / `"Alzheimer disease"`.

**The self-gating negative:**

**Ask:** gothic cathedrals + pubmed intent.

**Agent should:** run PubMed; if below floor / non-biomedical → **no**
`topicEvidence` block.

---

## 7. Honesty & edge cases (the part worth showing skeptics)

**In-copyright title → no full-text offer:**

**Ask:** "tomorrow and tomorrow and tomorrow" (title search, annotate).

**Agent should:** search + WikiData; OL probe finds no `ebook_access=public`.

- **Look for:** author bio OK; **`actions: []`**, no `openlibrary_ia`.

**Zero results:**

**Ask:** nonsense query `zxqwvk nonsense plurp`.

**Agent should:** search; return empty honestly.

- **Look for:** `resultCount: 0`, empty `records`, no invented hits.

**Fail-soft enrichment:** if WikiData/OL/PubMed is down, omit that annotation;
catalog results still present.

---

## 8. Everything at once

**Ask:** "Find books on CRISPR since 2020, just books — annotate them and show me
the recent medical evidence."

**Agent should:** HTTP search `CRISPR`, `type=Subject`, filters
`format:"Book"` + `publishDate:[2020 TO *]`, `limit=5`, annotate (WikiData + IA),
PubMed topic block, optional HTRC badges — all fail-soft.

- **Shows:** full pipeline in one turn.
- **Look for:** filtered results, per-record annotations where available,
  `topicEvidence` when PubMed qualifies.

---

## What to notice across all of these

- **Permalinks are real** catalog links; top-level `searchUrl` opens the whole
  search in the catalog UI.
- **Every full-text surface carries a `readUrl`** (Archive.org / Gutenberg /
  HathiTrust) so the user can read themselves, not only get an AI summary.
- **Missing annotations are honest absences**, not errors.
- **IA (public domain) + HTRC (in-copyright)** are complementary when features
  are obtainable; Lib-Skills never invents HTRC themes.
- Enrichment is **fail-soft and parallel**: one slow source never blocks a search.
- **No Python required** — agent HTTP steps replace Lib-Bot scripts while aiming
  for the same user-facing presentation.
