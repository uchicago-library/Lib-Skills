# Lib-Skills — Design

Lib-Skills is the **markdown-HTTP** twin of Lib-Bot’s `catalog-search` skill.
Same pipeline, same honesty rules, same user-facing envelope — different
execution substrate.

Behavior is derived from Lib-Bot’s Python enrichers and scripts
([uchicago-library/Lib-Bot](https://github.com/uchicago-library/Lib-Bot)), not
from memory. When heuristics diverge, Lib-Bot code wins; update this pack to
match.

---

## Markdown-HTTP vs scripts

| | Lib-Skills | Lib-Bot |
|---|---|---|
| Instructions | `SKILL.md` with explicit HTTP steps + heuristics | `SKILL.md` + `scripts/*.py` |
| Normalization | Agent maps VuFind JSON → schema | `catalog_search.normalize` |
| Consistency | Prompt discipline + schema section | Deterministic code |
| Token burn | Higher (acceptable) | Lower |
| Install | Global markdown skill | Repo + Python venv |
| What this pack ships | Markdown only | Scripts + optional `htrc-feature-reader` |

**Tradeoff accepted:** Cowork/Chat users get near-parity presentation without
git/Python/Code. Lib-Bot remains the authoritative Code path and the place to
harden edge cases in Python.

---

## Pipeline (unchanged from Lib-Bot)

```
query → SEARCH (VuFind) → FILTER/facets → ANNOTATE top-N → present → ACT on demand
```

Enrichment tiers match Lib-Bot:

- **Inline (cheap):** WikiData author fact; OpenLibrary→IA full-text badge;
  optional HathiTrust “content analysis available” badge; optional set-level
  PubMed `topicEvidence`.
- **On-demand (expensive):** IA/Gutenberg full-text pull; verify-first findtext;
  HTRC EF fingerprint when features exist.

---

## Fidelity bar

User-facing output must be **as close as possible** to the Lib-Bot scripts
version:

- Same badges (WikiData one-liner; 📖 full text available + `readUrl`; optional
  HathiTrust content-analysis badge)
- Same schema keys: `searchUrl`, `permalink`, `annotations.*`, `actions[]`,
  `topicEvidence?`, findtext `candidates[]`
- Same honesty / fail-soft / high-precision full-text badge
  (`ebook_access == public` only)
- Always offer `searchUrl` and human `readUrl` (not raw `source_url`)
- Higher token burn is OK
- Assume catalog API reachable; document VPN in README (ignore Anubis gating for
  design — treat it as an access prerequisite, not a product feature)

---

## What stays out of this pack

- No `scripts/`, no `pyproject.toml`, no venv as a requirement
- Do not modify or replace Lib-Bot
- No invented catalog JSON when VuFind is unreachable

---

## Conscious parity gaps

### HTRC analyze (primary gap)

Lib-Bot’s `analyze.py` uses `htrc-feature-reader` to download Extracted Features
(2025.04 stubbytree) and compute POS-filtered:

- `topThemes` — common nouns (`NN`/`NNS`)
- `topNames` — proper nouns (`NNP`/`NNPS`)

Lib-Skills **can** still:

- Run the HathiTrust **Bib API** join and attach the same inline badge shape
  (`contentAnalysis`, `htid`, `rights`, `readUrl`, `actions[].content_analysis`)
- Extract `htid` from a user-supplied HathiTrust URL
- Offer `readUrl` = `https://babel.hathitrust.org/cgi/pt?id={htid}`

Lib-Skills **must not**:

- Never invent `topThemes` or `topNames`
- Claim a vocabulary fingerprint when EF were not obtained

If the harness cannot obtain EF, say so honestly and stop — badge + `readUrl` +
rights are enough.

### Determinism

Without shared Python modules, agent implementations may vary slightly in
ranking/scan-pick edge cases. `SKILL.md` ports the same rules
(`clean_author_name`, library-scan rank, Gutenberg verify-first, etc.) so
behavior stays aligned.

### Deferred (same as Lib-Bot)

WorldCat, OpenSyllabus, FOLIO live availability — out of scope here too.

---

## Source of truth map

| Concern | Lib-Bot module | Lib-Skills section |
|---|---|---|
| Search + normalize | `catalog_search.py` | SKILL §1–2 |
| WikiData | `enrichers/wikidata.py` | SKILL §3a |
| OL→IA badge + pull | `enrichers/openlibrary_ia.py` | SKILL §3b, §4 |
| HathiTrust badge | `enrichers/hathitrust.py` `probe` | SKILL §3c |
| HTRC analyze | `enrichers/hathitrust.py` `analyze` | SKILL §5 (gap) |
| PubMed | `enrichers/pubmed.py` | SKILL §3d |
| Findtext | `enrichers/public_fulltext.py` | SKILL §4b |
| Fulltext CLI | `fulltext.py` | SKILL §4 |

See Lib-Bot [`DESIGN.md`](https://github.com/uchicago-library/Lib-Bot/blob/master/DESIGN.md)
for the full enricher contract and source roster.
