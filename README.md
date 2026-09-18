# Lib-Skills

A **pure-markdown** skill pack for University of Chicago Library catalog search
and enrichment — for Claude **Cowork / Chat** (and any harness that installs a
global markdown skill). **No git clone, no Python, no Code / terminal required.**

The first skill is **`catalog-search`**: search the Library catalog and enrich
results with WikiData author context, Internet Archive full-text badges,
verify-first Project Gutenberg / IA discovery, optional PubMed topic evidence,
and HathiTrust/HTRC content fingerprints — executed by the agent via HTTP.

Related (scripts / Code path): [Lib-Bot](https://github.com/uchicago-library/Lib-Bot).

**Jump to:**

- [Install (Claude Cowork / Chat)](#install-claude-cowork--chat)
- [Catalog access / VPN](#catalog-access--vpn)
- [What `catalog-search` does](#what-catalog-search-does)
- [Status](#status)

**Dig deeper:**

- [`DESIGN.md`](DESIGN.md) — markdown-HTTP design; fidelity & honesty; HTRC HTTP path; VPN
- [`skills/catalog-search/SKILL.md`](skills/catalog-search/SKILL.md) — agent skill
- [`skills/catalog-search/EXAMPLES.md`](skills/catalog-search/EXAMPLES.md) — demo tour

---

## Install (Claude Cowork / Chat)

1. Obtain this pack (clone or download once published as
   `uchicago-library/Lib-Skills`).
2. Zip the **`catalog-search`** folder (the directory that contains `SKILL.md` —
   i.e. `skills/catalog-search/`).
3. In Claude Cowork: **Customize → Skills → Upload zip**. Upload that zip so the
   harness loads `SKILL.md`.
4. No Python, no venv, no `config.json` file is required. Defaults are baked into
   the skill:

   - **`catalog_base`:** `https://catalog.lib.uchicago.edu/vufind`
   - **`probe_depth_n`:** `5`
   - **`contact_email`:** optional (add to User-Agent when calling public APIs)

5. Talk to the agent in natural language — it follows
   [`skills/catalog-search/SKILL.md`](skills/catalog-search/SKILL.md).

See [`EXAMPLES.md`](skills/catalog-search/EXAMPLES.md) for a full demo tour
(asks + expected shows / look-fors).

---

## Catalog access / VPN

The UChicago VuFind **Search & Record API** is IP-gated / challenge-gated
(campus allowlist; off-box callers may see Anubis/bot-check HTML or `403`).
Off-campus users typically need **VPN or campus network** for catalog search to
return JSON.

Enrichment sources (WikiData, OpenLibrary / Internet Archive, Project Gutenberg,
PubMed, HathiTrust Bib API) are **public internet** and work without VPN.

**This skill assumes the catalog API is reachable** when you use it. If a search
returns `403`, challenge HTML, or non-JSON, tell the user to connect via campus
VPN / network and retry — **do not invent holdings**.

---

## What `catalog-search` does

- **Search & filter** via VuFind `GET {catalog_base}/api/v1/search`, with
  `searchUrl` + per-record permalinks.
- **Inline annotate** top-N: WikiData author context + OpenLibrary→IA
  “full text available” badge (opt-in HathiTrust badge / PubMed topic evidence).
- **On demand:** pull IA/Gutenberg full text; verify-first findtext; HTRC-style
  content fingerprint **only when Extracted Features are obtainable**.
- **Honesty:** no invented enrichment; badges are high-precision
  (`ebook_access == public` only); fail-soft.

---

## Status

Initial pack for Brad / Tucker review. Intended publish target:
`uchicago-library/Lib-Skills` after approval (remote creation is separate).

`catalog-search` v1 treats badges/annotations as **first-class**. HTRC
fingerprints use stubbytree HTTPS (no venv); Cowork needs an off-context
bunzip/aggregate step — otherwise honest refusal. Never invent themes or names.
