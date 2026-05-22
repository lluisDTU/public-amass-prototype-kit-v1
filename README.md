# Amass Prototype Kit — v0.1.0

Amass-API-powered prototype kit for AI builders (Lovable, Claude, Cursor). Each `<slug>/SKILL.md` is a self-contained instruction sheet that scaffolds a working biomedical-evidence app on the [Amass API](https://platform.amass.tech) in one shot.

## What ships in v0.1.0

`amass-sr-pre-screen` — a systematic-review pre-screening tool that takes a curated PMID dump from an upstream PRISMA search (PubMed / Embase / CENTRAL / Scopus), applies the Amass trust-filter set (JuFo journal quality + retraction flag + citation count), and emits a Rayyan/Covidence-importable RIS file plus an audit-trail CSV — drops a ~5,000-PMID candidate set to ~100-500 papers before title/abstract screening.

## Build it in Lovable

Paste this 2-sentence build prompt into Lovable (or another AI builder that fetches build skills from URLs):

    Build a systematic-review pre-screening tool with the Amass API. Fetch your build skill: https://raw.githubusercontent.com/lluisDTU/public-amass-prototype-kit-v1/main/amass-sr-pre-screen/SKILL.md — credentials come from the Amass platform console at https://platform.amass.tech.

Lovable fetches the SKILL.md from the raw URL, scaffolds a Next.js + TypeScript app, and pre-fills it with a GLP-1 receptor agonists in obesity worked example. When prompted for credentials, paste your Amass API key from the [Amass Platform](https://platform.amass.tech).

## Roadmap

v0.1.0 is a staged-validation release shipping **one prototype** out of a 6-prototype kit. Subsequent versions ship the rest as each prototype passes its own empirical Lovable scaffold test (per the per-prototype publication patch pass discipline):

| Version | Prototype |
|---|---|
| v0.2.0 | `amass-regulatory-evidence-assembler` |
| v0.3.0 | `amass-retraction-cascade-monitor` |
| v0.4.0 | `amass-pipeline-monitor`, `amass-trust-filtered-rag`, `amass-claim-to-trial-verifier` |

## About Amass

The Amass API exposes two live Cores — BiomedCore (biomedical publications) + TrialCore (clinical trials) — with canonical Amass IDs (`AMBC_*` / `AMTC_*`), lookup endpoints from external identifiers (PMID / DOI / NCT), built-in trust filters (JuFo + retraction + citation count), and paper-to-trial cross-core graph edges that competitors do not expose as a single API surface. See [https://platform.amass.tech](https://platform.amass.tech) for credentials and developer documentation.

## License

Apache-2.0. Copyright 2026 Amass Technologies. See [LICENSE](./LICENSE).
