# Amass Prototype Kit — v0.4.0

Amass-API-powered prototype kit for AI builders (Lovable, Claude, Cursor). Each `<slug>/SKILL.md` is a self-contained instruction sheet that scaffolds a working biomedical-evidence app on the [Amass API](https://platform.amass.tech) in one shot.

## What ships in v0.4.0

v0.4.0 ships **all six prototypes** of the kit:

- `amass-sr-pre-screen` — a systematic-review pre-screening tool that takes a curated PMID dump from an upstream PRISMA search (PubMed / Embase / CENTRAL / Scopus), applies the Amass trust-filter set (JuFo journal quality + retraction flag + citation count), and emits a Rayyan/Covidence-importable RIS file plus an audit-trail CSV — drops a ~5,000-PMID candidate set to ~100-500 papers before title/abstract screening.
- `amass-regulatory-evidence-assembler` — an auditable literature-evidence assembler for FDA / EMA regulatory submissions (CTD Module 2.5 / 2.7 / PSUR). Resolves a curated PMID/DOI/NCT submission scope to canonical `AMBC_` / `AMTC_` IDs, walks the paper→trial cross-core edge per paper, and emits the audit-CSV row set anchoring each cited paper to its supporting trial via canonical `AMTC_` IDs that survive NCT-registry revisions between submission (t0) and approval (tN).
- `amass-pipeline-monitor` — a sponsor-watchlist pipeline-monitor CI agent. Takes a YAML sponsor watchlist + indication-cluster scope; runs a weekly digest that surfaces newly-registered Phase 2/3 trials per sponsor, walks each surfaced trial's `referencesBiomedCore` cross-core edge to anchor new published evidence, and flags retraction-flagged citations across the walked papers — all keyed by canonical Amass IDs (`AMTC_` / `AMBC_`) for reproducible week-over-week diffs. The 3-panel weekly dashboard (new trials / new papers / retraction-flagged) renders the trial→paper traversal end-to-end on one Amass call surface that on PubMed + ClinicalTrials.gov + OpenAlex would take 3-4 calls across separate entity-key systems. **Worked example:** SCLC-DLL3-2026Q2 watchlist (Amgen + Roche + AbbVie + Boehringer Ingelheim + Harpoon Therapeutics) anchored on Tarlatamab + DeLLphi-301 (NCT05060016) + PMID 37861218.
- `amass-trust-filtered-rag` — a trust-filtered biomedical RAG tool with the Amass API exposed as MCP-callable tools. Wraps BiomedCore endpoints as Model Context Protocol tools the LLM invokes mid-conversation, with built-in trust filters (JuFo journal quality + retraction flag + citation count) applied as default-on parameters. The chat surface answers biomedical questions by routing through Amass — every citation surfaces with `isRetracted`, `journalQualityJufo`, and `citationCount` rendered as trust badges, no hallucinated references. **Worked example:** Tarlatamab DeLLphi-301 primary publication (PMID 37861218 Ahn MJ et al. NEJM 2023).
- `amass-retraction-cascade-monitor` — a retraction-cascade dashboard for systematic-review and regulatory-affairs workflows. Resolves a curated bibliography to canonical `AMBC_` IDs, walks each paper's `references` + `citedBy` intra-core edges + `referencesTrialCore` cross-core edge in one Amass call per paper, and renders a 3-panel cascade dashboard surfacing inbound contamination (downstream citations of retracted work), outbound contamination (the retracted work's own references), and cross-core contamination (clinical trials the retracted paper describes). **Worked example:** Wakefield et al. Lancet 1998 PMID 9500320 (fully retracted 2010-02-06; verified via WebFetch against PubMed + Retraction Watch).
- `amass-claim-to-trial-verifier` — a claim-to-trial verification tool for publishers, editors, and regulatory affairs reviewers. Takes a marketing claim text + a DOI of the trial publication, walks DOI → `AMBC_` → cross-core to `AMTC_`, fetches the trial's registered primary endpoint, and surfaces a verdict card: `supported` / `not_supported` / `contradicts` / `needs_review` — with the claim text + trial primary outcome rendered verbatim alongside the verdict reasoning. **Worked example:** Tarlatamab "improves overall survival" claim + DOI 10.1056/NEJMoa2307980 → NCT05060016 (DeLLphi-301, primary endpoint = ORR per RECIST 1.1, not Overall Survival) → expected verdict: `needs_review` (category mismatch).

## Build it in Lovable

Paste the build prompt for your chosen prototype into Lovable (or another AI builder that fetches build skills from URLs). Lovable fetches the SKILL.md from the raw URL, scaffolds a TypeScript app (TanStack Start primary; Next.js App Router alternative — both work end-to-end), and pre-fills it with a verified worked example. When prompted for credentials, paste your Amass API key from the [Amass Platform](https://platform.amass.tech).

### amass-sr-pre-screen

    Build a systematic-review pre-screening tool with the Amass API. Fetch your build skill: https://raw.githubusercontent.com/lluisDTU/public-amass-prototype-kit-v1/main/amass-sr-pre-screen/SKILL.md — credentials come from the Amass platform console at https://platform.amass.tech.

Pre-filled with a GLP-1 receptor agonists in obesity illustrative SR scope.

### amass-regulatory-evidence-assembler

    Build an auditable regulatory-evidence assembler with the Amass API. Fetch your build skill: https://raw.githubusercontent.com/lluisDTU/public-amass-prototype-kit-v1/main/amass-regulatory-evidence-assembler/SKILL.md — credentials come from the Amass platform console at https://platform.amass.tech.

Pre-filled with a Tarlatamab (Imdelltra) / Amgen BLA 761344-dlle worked example.

### amass-pipeline-monitor

    Build a sponsor-watchlist pipeline-monitor CI agent with the Amass API. Fetch your build skill: https://raw.githubusercontent.com/lluisDTU/public-amass-prototype-kit-v1/main/amass-pipeline-monitor/SKILL.md — credentials come from the Amass platform console at https://platform.amass.tech.

Pre-filled with a SCLC-DLL3-2026Q2 watchlist (Amgen + Roche + AbbVie + Boehringer Ingelheim + Harpoon Therapeutics) anchored on Tarlatamab + DeLLphi-301 (NCT05060016) + PMID 37861218.

### amass-trust-filtered-rag

    Build a trust-filtered biomedical RAG tool with the Amass API as MCP tools. Fetch your build skill: https://raw.githubusercontent.com/lluisDTU/public-amass-prototype-kit-v1/main/amass-trust-filtered-rag/SKILL.md — credentials come from the Amass platform console at https://platform.amass.tech.

Pre-filled with the Tarlatamab DeLLphi-301 primary publication (PMID 37861218 Ahn NEJM 2023).

### amass-retraction-cascade-monitor

    Build a retraction-cascade monitor with the Amass API. Fetch your build skill: https://raw.githubusercontent.com/lluisDTU/public-amass-prototype-kit-v1/main/amass-retraction-cascade-monitor/SKILL.md — credentials come from the Amass platform console at https://platform.amass.tech.

Pre-filled with the Wakefield Lancet 1998 PMID 9500320 retracted-paper worked example.

### amass-claim-to-trial-verifier

    Build a claim-to-trial verifier with the Amass API. Fetch your build skill: https://raw.githubusercontent.com/lluisDTU/public-amass-prototype-kit-v1/main/amass-claim-to-trial-verifier/SKILL.md — credentials come from the Amass platform console at https://platform.amass.tech.

Pre-filled with a Tarlatamab "improves overall survival" claim + DeLLphi-301 NCT05060016 (expected verdict: `needs_review` — category mismatch).

## Status

v0.4.0 completes the kit — all 6 prototypes shipped to `main` as fetchable SKILL.md files. Future work iterates on:

- Per-prototype Lovable empirical-test feedback (patch passes published as minor-version bumps)
- New prototypes as new ICPs / use cases emerge (the kit can grow beyond 6)
- Canonical promotion to `amass-technologies/public-amass-prototype-kit` as the v1.0 gate

## About Amass

The Amass API exposes two live Cores — BiomedCore (biomedical publications) + TrialCore (clinical trials) — with canonical Amass IDs (`AMBC_*` / `AMTC_*`), lookup endpoints from external identifiers (PMID / DOI / NCT), built-in trust filters (JuFo + retraction + citation count), and paper-to-trial cross-core graph edges that competitors do not expose as a single API surface. See [https://platform.amass.tech](https://platform.amass.tech) for credentials and developer documentation.

## License

Apache-2.0. Copyright 2026 Amass Technologies. See [LICENSE](./LICENSE).
