# Lib-Skills — Design

Lib-Skills is a **markdown-HTTP** skill pack for University of Chicago Library
catalog search and enrichment. Agents follow explicit HTTP steps and heuristics
in `SKILL.md` — no Python, scripts, or venv.

---

## Markdown-HTTP execution

| Concern | How Lib-Skills does it |
|---|---|
| Instructions | `SKILL.md` with explicit HTTP steps + heuristics |
| Normalization | Agent maps VuFind JSON → schema in `SKILL.md` |
| Consistency | Prompt discipline + schema section |
| Token burn | Higher (acceptable for Cowork / Chat installs) |
| Install | Global markdown skill (zip upload) |
| What this pack ships | Markdown only |

**Tradeoff accepted:** Cowork/Chat users get catalog search + enrichment without
git/Python/Code setup.

---

## Pipeline

```
query → SEARCH (VuFind) → FILTER/facets → ANNOTATE top-N → present → ACT on demand
```

Enrichment tiers:

- **Inline (cheap):** WikiData author fact; OpenLibrary→IA full-text badge;
  optional HathiTrust “content analysis available” badge; optional set-level
  PubMed `topicEvidence`.
- **On-demand (expensive):** IA/Gutenberg full-text pull; verify-first findtext;
  HTRC EF fingerprint when features exist.

---

## Fidelity & honesty

User-facing output must follow this pack’s schema and honesty rules:

- Badges (WikiData one-liner; 📖 full text available + `readUrl`; optional
  HathiTrust content-analysis badge)
- Schema keys: `searchUrl`, `permalink`, `annotations.*`, `actions[]`,
  `topicEvidence?`, findtext `candidates[]`
- Honesty / fail-soft / high-precision full-text badge
  (`ebook_access == public` only)
- Always offer `searchUrl` and human `readUrl` (not raw `source_url`)
- Higher token burn is OK
- Assume catalog API reachable; document VPN in README (ignore Anubis gating for
  design — treat it as an access prerequisite, not a product feature)

---

## What stays out of this pack

- No `scripts/`, no `pyproject.toml`, no venv as a requirement
- No invented catalog JSON when VuFind is unreachable

---

## HTRC analyze (HTTP path; no venv)

HTRC Extracted Features (EF) 2025.04 files are public over HTTPS at stubbytree
paths under
`https://data.analytics.hathitrust.org/features-2025.04/…/*.json.bz2`. Detailed
stubbytree download + POS aggregation live in
`skills/catalog-search/references/htrc-ef.md`; `SKILL.md` §5 keeps when-to-use,
htid resolution, honesty/fail-soft, and report shape.

**Works in Cowork when** the harness can download bz2, decompress, and aggregate
token/POS counts **off-context**. Chat-only / no sandbox → honest refusal +
`readUrl` / Bib-API rights / optional metadata URL.

Lib-Skills **must not**:

- Invent `topThemes` / `topNames` or guess pairtree / fake EF URLs
- Claim a fingerprint when EF were not obtained

### Determinism

Without shared code modules, agent implementations may vary slightly in
ranking/scan-pick edge cases. `SKILL.md` spells out the same rules
(`clean_author_name`, library-scan rank, Gutenberg verify-first, etc.) so
behavior stays aligned.

### Deferred

WorldCat, OpenSyllabus, FOLIO live availability — out of scope.

---

## EXAMPLES.md (human testers only)

`skills/catalog-search/EXAMPLES.md` is a **manual demo tour for human testers**
verifying the skill in Cowork/Chat. It is **not** agent instructions and is not
required reading for Claude. Zip it with the skill only if useful for humans;
Claude loads `SKILL.md` (+ `references/`).

---

## Skill map

| Concern | Lib-Skills section |
|---|---|
| Search + normalize | SKILL §1–2 |
| WikiData | SKILL §3a |
| OL→IA badge + pull | SKILL §3b, §4 |
| HathiTrust badge | SKILL §3c |
| HTRC analyze | SKILL §5 + `references/htrc-ef.md` |
| PubMed | SKILL §3d |
| Findtext | SKILL §4b |
| Fulltext pull | SKILL §4 |
