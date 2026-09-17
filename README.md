# Lib-Skills

A **pure-markdown** skill pack for University of Chicago Library catalog search
and enrichment — for Claude **Cowork / Chat** (and any harness that installs a
global markdown skill). **No git clone, no Python, no Code / terminal required.**

The first skill is **`catalog-search`**: search the Library catalog and enrich
results with the same badges, annotations, honesty rules, and result schema as
the scripts-backed [Lib-Bot](https://github.com/uchicago-library/Lib-Bot) skill —
executed by the agent via HTTP instead of local Python.

**Jump to:**

- [Lib-Skills vs Lib-Bot](#lib-skills-vs-lib-bot)
- [Install (Claude Cowork / Chat)](#install-claude-cowork--chat)
- [Catalog access / VPN](#catalog-access--vpn)
- [Status](#status)

**Dig deeper:**

- [`DESIGN.md`](DESIGN.md) — markdown-HTTP vs scripts; fidelity bar; HTRC HTTP path
- [`skills/catalog-search/SKILL.md`](skills/catalog-search/SKILL.md) — agent skill
- [`skills/catalog-search/EXAMPLES.md`](skills/catalog-search/EXAMPLES.md) — demo tour

---

## Lib-Skills vs Lib-Bot

| | **Lib-Skills** (this pack) | **Lib-Bot** |
|---|---|---|
| Audience | Cowork / Chat users installing a global markdown skill | Claude Code (and other agent harnesses) with a repo + venv |
| Runtime | Agent performs HTTP steps described in `SKILL.md` | `${LIBBOT_PYTHON} skills/…/scripts/*.py` |
| Install | Copy / enable the skill markdown — no clone, no pip | `git clone` + `python3 -m venv` + `pip install` |
| Fidelity bar | User-facing output ≈ scripts version (same badges, schema, honesty) | Source of truth for behavior; scripts guarantee consistency |
| Token cost | Higher (agent reasons through HTTP) — acceptable | Lower (scripts normalize + probe) |
| HTRC analyze | Stubbytree HTTPS + POS aggregate when harness can bunzip off-context; else honest refusal — never invent | `htrc-feature-reader` in venv (convenience only) |

**Lib-Bot remains the Code path.** Keep its git/Python/scripts intact. This pack
is a separate product name (**Lib-Skills**) so Cowork/Chat users get presentation
parity without a developer setup. It does **not** replace Lib-Bot.

---

## Install (Claude Cowork / Chat)

1. Obtain this pack (clone or download once published as
   `uchicago-library/Lib-Skills`, or use a shared skill folder).
2. Install **`skills/catalog-search`** as a global skill for Claude Cowork/Chat
   (or your harness’s equivalent of “install this markdown skill”). Point the
   harness at that directory so it loads `SKILL.md`.
3. No Python, no venv, no `config.json` file is required. Defaults are baked into
   the skill:

   - **`catalog_base`:** `https://catalog.lib.uchicago.edu/vufind`
   - **`probe_depth_n`:** `5`
   - **`contact_email`:** optional (add to User-Agent when calling public APIs)

4. Talk to the agent in natural language — it follows
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

## What `catalog-search` does (parity with Lib-Bot)

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
**Not** a fork that replaces Lib-Bot — complementary install path for
markdown-only harnesses.

`catalog-search` v1 treats badges/annotations as **first-class**. Presentation
target = Lib-Bot scripts output. HTRC fingerprints use stubbytree HTTPS (no
venv); Cowork needs an off-context bunzip/aggregate step — otherwise honest
refusal. Never invent themes or names.
