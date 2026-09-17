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

### HTRC analyze (HTTP path; no venv)

Lib-Bot’s `analyze.py` uses `htrc-feature-reader` as a convenience. The same
2025.04 EF files are public over HTTPS at stubbytree paths under
`https://data.analytics.hathitrust.org/features-2025.04/…/*.json.bz2`. Lib-Skills
documents that recipe + POS aggregation in `SKILL.md` §5 (no Python package).

**Works in Cowork when** the harness can download bz2, decompress, and aggregate
token/POS counts **off-context**. Chat-only / no sandbox → honest refusal +
`readUrl` / Bib-API rights / optional metadata URL.

Lib-Skills **must not**:

- Invent `topThemes` / `topNames` or guess pairtree / fake EF URLs
- Claim a fingerprint when EF were not obtained

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
| HTRC analyze | `enrichers/hathitrust.py` `analyze` | SKILL §5 (stubbytree HTTPS) |
| PubMed | `enrichers/pubmed.py` | SKILL §3d |
| Findtext | `enrichers/public_fulltext.py` | SKILL §4b |
| Fulltext CLI | `fulltext.py` | SKILL §4 |

See Lib-Bot [`DESIGN.md`](https://github.com/uchicago-library/Lib-Bot/blob/master/DESIGN.md)
for the full enricher contract and source roster.
