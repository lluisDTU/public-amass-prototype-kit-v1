---
name: amass-sr-pre-screen
description: Use when building a PRISMA pre-screen credibility-filter for systematic reviews (Cochrane / SR-as-a-service / academic). Provides build instructions for batch-resolving a curated PMID dump to canonical Amass IDs, post-filtering on JuFo + retraction + citation count, and emitting a Rayyan/Covidence-importable RIS file with an audit-trail CSV in a TypeScript app (Lovable typically scaffolds TanStack Start; Next.js App Router is the alternative path — both work end-to-end).
license: Apache-2.0
metadata:
  author: amass
  version: "0.1.1"
---

# SR Pre-Screen Skill (Amass BiomedCore)

Build a working PRISMA pre-screen credibility-filter on the Amass API. The user is a systematic-review researcher running PRISMA 2020 (per `02-research/personas-and-icp.md:57-82`) who just got Amass API credentials and wants a TypeScript pre-screen tool running locally in under five minutes. Lovable typically scaffolds TanStack Start (TanStack Router + `createServerFn` server-functions); Next.js (App Router with `app/api/<endpoint>/route.ts`) is the alternative path. Both work end-to-end against the same `lib/amass.ts` client pattern — the Amass primitives are stack-agnostic.

The SR researcher pastes a curated PMID dump from the upstream PRISMA search step (PubMed / Embase / CENTRAL / Scopus produce the standard 5,000-PMID export); the tool resolves each PMID to a canonical `AMBC_` Amass ID via batch lookup, fans out per-paper GETs to retrieve `isRetracted` + `journalQualityJufo` + `citationCount`, post-filters client-side against the configured credibility thresholds, and emits a Rayyan/Covidence-importable RIS file plus an audit-trail CSV — dropping the 5,000-PMID candidate set to a ~100-500-paper screening set before title/abstract screening. The worked example binds to an illustrative SR scope ("GLP-1 receptor agonists in obesity") with ~10 representative PMIDs. `[identifier-verification caveat — example identifiers assembled without live web access; re-verify against PubMed before binding to a real SR workflow]`

---

## Step 1 — Confirm the build

Use `AskUserQuestion` if available; otherwise ask the user directly before writing any code.

Fetch https://api.amass.tech/docs/biomedcore `[TODO: confirm Amass docs URL]` first, then ask one question. Skip / "you decide" → use the default.

**Which credibility-filter thresholds should the pre-screen apply?** _(default: `min_jufo: 2` + `allow_retracted: false` + `min_citation_count: 5`)_

- **`min_jufo: 2`** — require JuFo tier 2 or 3 (domain-leading or highest) per `01-capabilities/capability-map.md:135`; drops papers from journals JuFo flags as 0 or 1 *(default)*
- **`allow_retracted: false`** — drop retracted papers per `isRetracted` field; the conservative PRISMA pre-screen default *(default)*
- **`min_citation_count: 5`** — drop papers with `citationCount < 5` as a conservative post-filter signal *(default)*
- `min_jufo: 3` — stricter; require JuFo tier 3 only (highest tier)
- `min_jufo: 0` — opt out of the JuFo leg entirely (US-buyer-friendly; JuFo is a Finnish national scale per `01-capabilities/capability-map.md:164`)
- `include_referencesTrialCore: true` — opt-in cross-core walk; for SRs whose inclusion criteria intersect literature with a target trial set

**`include_referencesTrialCore: true` opts into a paper-side cross-core include.** The v0.1 fetches `include=referencesTrialCore` on each per-paper GET when this opt-in is set; the audit-trail CSV gains a `referencesTrialCore_AMTCs` column. At-scale symmetry of the cross-core spine is `[doc-silent / not yet a published guarantee]` per E-007 — empty arrays render as honest emptiness (review-article papers commonly have no cited trials); see Hard Rule #12.

If the user supplies non-default thresholds, generate matching "Try sample" seed prompts so every wired threshold gets exercised against the GLP-1 RA worked-example PMID set.

---

## Stack — flexible at the framework layer, principle-pinned at the API layer

**TypeScript + React + a server-function-capable framework.** Lovable defaults to **TanStack Start** (`@tanstack/react-router` + `@tanstack/react-start`'s `createServerFn` for server-side handlers). **Next.js (App Router)** with `app/api/<endpoint>/route.ts` POST handlers is the equivalent alternative path. Both work end-to-end against the same `lib/amass.ts` client; pick whichever your AI builder produces by default. The Amass primitives (canonical-ID resolution + per-item-error semantics + cross-core walks + trust filters) are stack-agnostic — the framework only wraps them.

What IS pinned (load-bearing):

```json
"@amass/sdk":             "{{amass-sdk-version}}",
"react":                  "{{react-version}}",
"react-dom":              "{{react-dom-version}}",
"typescript":             "{{typescript-version}}",
"zod":                    "{{zod-version}}",
"lucide-react":           "{{lucide-react-version}}",
"server-only":            "{{server-only-version}}"
```

What's framework-specific (one set OR the other, not both):

- **TanStack Start path**: `@tanstack/react-router`, `@tanstack/react-start`, `vite`, `@vitejs/plugin-react`.
- **Next.js (App Router) path**: `next` (latest stable), `@types/node`.

Pin as **exact versions** (no `^`). `@amass/sdk` is pinned exactly because the audit-trail CSV `lookup_error` column semantics depend on the per-item-error array shape returned by `POST /records/lookup` per `01-capabilities/capability-map.md:121` — any SDK version that paraphrases or restructures the per-item-error string breaks the load-bearing UV-6 / A5 distinctive the prototype demonstrates. The 5,000-PMID-input scale exercises this surface load-bearingly: at ~50 batch-lookup calls per pre-screen run, every per-item `error` string the upstream surfaces feeds the SR researcher's audit trail verbatim. `zod` is pinned because the input-validator schema (`PreScreenSchema`) IS the request contract — silent zod-version drift in default-value or refine semantics shifts what the route accepts. `server-only` is pinned because it's the build-time enforcement Hard Rule #2 depends on (works identically in Next.js and TanStack Start).

**Resolve `{{version}}` placeholders before `npm install`** — per Hard Rule #8 (ground-truth discipline), per-prototype dispatch reads installed `node_modules/<pkg>/package.json` for actual versions. A literal `{{next-version}}` or `{{tanstack-router-version}}` in `package.json` triggers `npm ERR! EINVALIDTAGNAME` and aborts the install.

**Two notes for AI builders that may produce either path.** (1) The route file location differs: TanStack Start uses `src/lib/<name>.functions.ts` with `createServerFn({ method: "POST" }).inputValidator(...).handler(...)`; Next.js App Router uses `app/api/<endpoint>/route.ts` with `export async function POST(req: Request)`. The reference snippet below is shown in Next.js shape — if your builder produces TanStack Start, translate the `POST(req)` handler to `createServerFn(...).handler(async ({ data }) => ...)` and the `req.json()` body parse to `.inputValidator((input) => PreScreenSchema.parse(input))`. (2) The page file location differs: TanStack Router uses `src/routes/index.tsx` with `createFileRoute("/")`; Next.js App Router uses `app/page.tsx`. Either way, the page calls into the server function / route handler the same way at the network level.

---

## What the app should do

The primary analyst-visible payoff is the **credibility-filtered RIS file** — importable into Rayyan or Covidence as the title/abstract screening corpus, dropping a 5,000-PMID PRISMA upstream-search dump to a ~100-500-paper screening set. The load-bearing Amass primitive is the trust-filter conjunction at non-commercial tier per UV-4 at `03-positioning/unique-values.md:114` — `isRetracted` as a server-side default field on `GET /records/{amassId}` per `capability-map.md:169` AND `journalQualityJufo` as a client-side post-filter on the returned field per `capability-map.md:164` (the JuFo Nordic/EU national-evaluation tier is distinctive — not exposed by PubMed natively, not exposed by OpenAlex) AND `citationCount` as a client-side post-filter on the returned field per the `quality-gate-papers` skill's documented MCP-vs-HTTP gap. The composition in one workflow against a single canonical-AMBC-ID record is the residual moat.

The input surface is a **paste-PRISMA-export textarea** — one PMID per line, matching the standard PubMed `Send to → File → PMID` export format that PRISMA upstream searches produce (PubMed / Embase / CENTRAL / Scopus per `personas-and-icp.md:61`). The empty-state "Try sample" loads the GLP-1 RA in obesity illustrative SR scope verbatim — `~10 representative PMIDs for the GLP-1 receptor agonist in obesity literature` (tooltip: "GLP-1 RA in obesity SR pre-screen — illustrative scope; small representative PMID set. `[identifier-verification caveat — example identifiers assembled without live web access; re-verify against PubMed before binding to a real SR workflow]`"). A "thresholds" form sits next to the textarea showing the active `min_jufo` + `allow_retracted` + `min_citation_count` values (tunable per Hard Rule #10).

The pre-screen run fans out batch-lookups + per-paper GETs at the 60 req / 60 s ceiling per `capability-map.md:383`. A 5,000-PMID input chunks into ~50 batch-lookup calls (~100 items each per `quality-gate-papers` SKILL.md usage warning and the conservative E-006 `items[]`-ceiling heuristic) + ~500 per-paper GETs at the post-filter step = ~550 calls per pre-screen run; wall-time ~15 min at the rate-limit ceiling. The route wraps the per-paper fan-out async generator in `withIdleTimeout(gen, 180_000)` per Hard Rule #7 and the client renders an analyst-visible **Cancel** button next to the primary "Run pre-screen" action while the fan-out is in flight. A progress indicator increments per resolved paper ("pre-screen run in progress — N of M papers post-filtered"); a stalled Amass call past the 180s idle threshold surfaces a "stream stalled — an Amass call likely hung" error without locking the UI indefinitely.

The final state offers two downloadable artifacts side-by-side: a **Rayyan/Covidence-importable RIS file** (`TY=JOUR, PMID, TI, AU, JO, JF, PY, DO, AB` per the RIS field mapping the screening tools expect) emitted from the filtered set, and an **audit-trail CSV** with the 11-column shape `pmid, accessed_at, AMBC_id, isRetracted, journalQualityJufo, citationCount, lookup_error, included_in_prescreen, threshold_min_jufo, threshold_min_citation_count, threshold_allow_retracted` — the three trailing `threshold_*` columns repeat the run-time threshold values per row so a reviewer downstream can reproduce the inclusion/exclusion verdicts without needing to remember what the screener set. The v0.1 returns the assembled rows in-memory in a single consolidated response — there is no monthly re-audit cron (JTBD-8 retraction-monitoring lands under "Want to extend it?" in Hand-off), no v1.0 BiomedCore-search-driven discovery (gated on E-025/E-026 per AP-1 / `anti-pitches.md:37`), no Python CLI sidecar. Render PMIDs / DOIs / `AMBC_` IDs in IBM Plex Mono; prose in Inter (Brand reference). Use Lucide React for icons. Lovable produces the UI shape from this spec — variability across builders is fine; the load-bearing constraint is that the four paragraphs above demonstrate the trust-filter conjunction at non-commercial tier, the lookup-then-fetch canonical-ID flow, the per-item-error column split, and the worked-example anchor.

---

## Hard rules

The 8 Amass-universal rules below are inherited verbatim across all 6 prototype SKILL.mds. Per-prototype rules 9-12 cover the load-bearing UX / error / discipline surfaces specific to brief 07.

1. **Credentials never enter committed code.** Generate `.env` and `.gitignore` it. Only ONE secret is asked of the user at hand-off:
   ```env
   AMASS_API_KEY=your_amass_api_key_here
   ```
   Per `CLAUDE.md` API conventions: the key reads from `AMASS_API_KEY`. The base URL is **hardcoded** to `https://api.amass.tech` in `lib/amass.ts` (single production base; no per-user variance). To point at staging or a local proxy for development, edit the `AMASS_API_BASE_URL` constant in `lib/amass.ts` directly — do NOT re-introduce an env var for this (adding `process.env.AMASS_API_BASE_URL` to `lib/amass.ts` causes Lovable's secrets-UX to prompt the user for a value they don't have, which is unnecessary friction). Never hardcode the API key, never log it, never commit it.

2. **Server-only `AmassClient`.** Lives in `lib/amass.ts` with `import "server-only"` at the top. Never imported from a `"use client"` component. The browser only ever speaks to `/api/<endpoint>`. The Bearer token must never reach the browser bundle; the `server-only` package enforces this at build time.

3. **Singleton/cached `AmassClient`.** Created once on server boot via `getAmassClient()` factory, cached via `clientPromise ??=` pattern. Per-request instantiation is wasted work; the Bearer header is computed once at module-import time and reused.

4. **`Authorization: Bearer ${AMASS_API_KEY}` on every Amass request.** Per `01-capabilities/capability-map.md` shared-auth section. On `401` / `403`, surface the credential error directly — retry won't fix it. On `429`, read `Retry-After` and back off exponentially (per the `lib/amass.ts` snippet below). Rate limit is **60 requests / 60 seconds, per user+org** — the per-call retry helper IS the rate-limit defence for the 5,000-PMID-driven ~550-call pre-screen fan-out.

5. **Read from the `data` key; handle errors at top-level AND per-item.** Per `capability-map.md:121`. Two distinct error shapes: top-level (any non-2xx) — `{"error": {"status": ..., "code": ..., "message": ...}}` — caught by the route's outer `try/catch`. Per-item (`POST /records/lookup` returns HTTP 200) — each array element is **either** `{"amassIds": ["AMBC_..."]}` **or** `{"error": "Not found"}`. The `mapLookupResult` helper preserves index alignment with the input items. **Never assume HTTP 200 means every item succeeded.**

6. **Lookup endpoints translate PMID/DOI/NCT → canonical `AMBC_`/`AMTC_` before fetching by ID.** Per `capability-map.md` §biomedcore-batch-lookup / §trialcore-batch-lookup. Raw HTTP `GET /records/{id}` accepts **only** canonical Amass IDs — passing a PMID directly returns 404. The workflow is always: `POST /records/lookup` with `[{pmid: "..."}]` → read `amassIds[0]` (handling per-item-error per Rule #5) → `GET /records/{amassId}`. The `lib/amass.ts` `resolvePmid` / `resolveDoi` / `resolveNct` helpers encode this discipline; for the SR pre-screen the batch form (`batchLookupBiomed`) is used directly with chunks of ~100 PMIDs.

7. **Idle-timeout + analyst-visible cancel button on long-running fan-outs.** Per `handoff-2026-05-21-workstream-C-prototype-kit.md` Section 6. Three requirements: (a) the route wraps any fan-out async generator in `withIdleTimeout(gen, 180_000)` — if no chunk arrives for 180s, the wrapper throws; (b) the client renders an analyst-visible **Cancel** button next to the primary action while a fan-out is in flight; (c) the route wraps body parse + client setup in an outer `try/catch` returning `Response.json({error}, {status: 500})`. Without all three, a single stalled Amass call locks the UI indefinitely on a 15-minute pre-screen job.

8. **Ground-truth discipline.** Every URL, type, function name, version, and config shape in this SKILL.md must come from a source the agent can read — a fetched doc page, a `.d.ts` in `node_modules/@amass/sdk/`, the installed package's exports, or a verbatim quote from `01-capabilities/capability-map.md`. If you can't point to a source, don't write it. Forbidden: fabricating endpoints; guessing field names; `as unknown as ...` or `as any` on `@amass/sdk` values; ignoring `tsc` errors. When the runtime contradicts what's written here, fix the cause.

### Per-prototype hard rules

9. **Caller-supplied PRISMA upstream-search input.** The tool consumes a curated PMID list (typically a PubMed / Embase / CENTRAL / Scopus export from the PRISMA upstream-search step per `personas-and-icp.md:61`); it does NOT replace the upstream-search step. SR researchers run PRISMA upstream discovery themselves via the standard PRISMA tools as part of PRISMA 2020 protocol — the tool slots **between** the upstream-search step (which produces the 5,000-PMID dump this brief consumes) and the title/abstract screen step (Rayyan / Covidence / Elicit per `personas-and-icp.md:64`, which consumes the credibility-filtered RIS this brief produces). Rationale: keeps v0.1 within doc-published guarantees and explicitly out of AP-1 trespass — v0.1 does NOT invoke BiomedCore search (`GET .../records?query=...`); the v1.0 in-tool search-driven discovery layer is `[lifecycle: post-fix-only]` per brief 07 line 53 + `anti-pitches.md:37` until E-025/E-026 closes and ships only then.

10. **Trust-filter post-filter discipline (`quality-gate-papers` MCP-vs-HTTP gap).** Three of the three trust signals come back on the per-paper GET, but they reach the filter at different layers — and conflating them collapses the brief's load-bearing rubric mandatory-cut compliance. `isRetracted` is a server-side default field on `GET /records/{amassId}` per `capability-map.md:169` (the default-field behaviour, not a query parameter). `journalQualityJufo` and `citationCount` are returned-field client-side post-filters per the `quality-gate-papers` SKILL.md's documented MCP-vs-HTTP gap warning — server-side `minCitationCount` is `[MCP-unavailable]` per `capability-map.md:134` AND is a mandatory cut on the rubric's server-side-claim rule per `_scoring-rubric.md:44, 199`. The audit-trail CSV records all three trust signals verbatim from the live record so the SR researcher can re-tune the thresholds post-hoc; never invent JuFo or citation values from PubMed-side metadata. Rationale: the brief's defensibility audit holds because v0.1 routes around AP-1 via the lookup-then-fetch + client-side-post-filter path — the post-filter discipline IS the AP-1-avoidance mechanism.

11. **Per-item error verbatim preservation.** Surface the upstream-generated per-item error string verbatim in the `lookup_error` column — never paraphrased, never elided, never normalised. The `mapLookupResult` helper in `lib/amass.ts` enforces this by passing the raw `r.error` string through unchanged. Rationale: for an SR audit trail (per PRISMA 2020 reporting standards + the PMC10982619 evidence-synthesis paper at `personas-and-icp.md:71` recommending "checking retraction status just prior to publication"), "this PMID is unresolved with reason X" vs "this PMID is silently absent" is the difference between an auditable gap and an audit failure; paraphrasing the upstream reason string collapses the two and breaks the load-bearing UV-6 / A5 distinctive against OpenAlex (matched-only `filter=ids.pmid:X|Y|Z` with silent omission per `02-research/alternatives-matrix.md`).

12. **Worked-example anchoring (MEDIUM tier, caveat-and-ship).** Per `.claude/skills/worked-example-anchoring/SKILL.md` section (d) line 86 — Prototype 3 is **MEDIUM**, caveat-and-ship. The workflow is a screening-recommendation surface (the tool emits a candidate set for downstream title/abstract screening) — NOT a defamation-adjacent assertion surface (the tool does not assert any specific PMID is retracted; it displays the live `isRetracted` field from the Amass record verbatim alongside the credibility-filter decision). The "Try sample" loads the GLP-1 RA in obesity illustrative SR scope with ~10 representative PMIDs covering the literature shape (RCTs of semaglutide, liraglutide, tirzepatide in obesity / weight loss endpoints). Carry the caveat marker `[identifier-verification caveat — example identifiers assembled without live web access; re-verify against PubMed before binding to a real SR workflow]` at three anchor sites: the title intro paragraph, the Try-sample tooltip, and the Hand-off verification preamble. `isRetracted` + JuFo + citation count columns populate from the **live Amass record at runtime**, never from a hardcoded fixture — the display=claim defamation-adjacent guard structurally holds (Lovable scaffolds against the live API, not against fabricated retraction data). Rationale: the brief's MEDIUM tier permits caveat-and-ship because the workflow is screening-recommendation (analyst makes the final inclusion decision in Rayyan/Covidence downstream), not retraction-claim-assertion; the live-Amass-binding requirement preserves the structural defamation-adjacent guard while the caveat marker keeps the verification-stakes audit visible.

---

## Brand reference

Source of truth for the Amass app look-and-feel. Load before writing `globals.css`.

### Logo

Render the Amass wordmark at the **top-left of the header**, ~24-32px, as an `<img>` (don't bundle locally). Place it left of the page title.

- SVG: `{{amass-logo-svg-url}}`
- PNG: `{{amass-logo-png-url}}`

`[TODO: confirm Amass logo URLs with the Amass design team]`

**Wordmark fallback.** If the SVG URL is not yet resolved (the TODO flag above), render the text **`amass`** in Inter weight 800 24px as the wordmark fallback. Do not ship a broken `<img>` tag with the literal placeholder; the text fallback is the visible artifact during the deployment-prep phase.

### Typography

- **Inter** — UI text, labels, headings, body. Weights 400/600/700/800/900.
- **IBM Plex Mono** — numerics, IDs (PMID / DOI / `AMBC_` / `AMTC_`), codes, monospace inputs. Weights 400/500/700. Never for prose.

Load both from Google Fonts (`https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700;800;900&family=IBM+Plex+Mono:wght@400;500;700&display=swap`).

### CSS tokens

Add to `globals.css`. Dark mode follows the OS via `prefers-color-scheme` — no toggle, no `next-themes`, no class swap.

```css
:root {
  --background: 0 0% 100%;
  --foreground: 0 0% 6%;
  --card: 0 0% 100%;
  --card-foreground: 0 0% 6%;
  --popover: 0 0% 100%;
  --popover-foreground: 0 0% 6%;
  --primary: 0 0% 6%;
  --primary-foreground: 0 0% 100%;
  --secondary: 0 0% 100%;
  --secondary-foreground: 0 0% 6%;
  --accent: 0 0% 98%;
  --accent-foreground: 222 47% 11%;
  --muted: 0 0% 98%;
  --muted-foreground: 0 0% 41%;
  --destructive: 0 84% 60%;
  --destructive-foreground: 0 0% 100%;
  --border: 0 0% 93%;
  --input: 0 0% 90%;
  --ring: 215 16% 47%;
  --radius: 0.5rem;
  --variant-error-bg: 0 100% 97%;
  --variant-error-border: 0 93% 86%;
  --variant-error-text: 0 72% 37%;
  --amass-accent: 215 80% 50%;             /* placeholder HSL; TODO: confirm Amass accent token with design team */
  --amass-accent-foreground: 0 0% 100%;    /* placeholder HSL; TODO: confirm Amass accent-foreground token with design team */
}

@media (prefers-color-scheme: dark) {
  :root {
    --background: 0 0% 6%;
    --foreground: 0 0% 90%;
    --card: 0 0% 10%;
    --card-foreground: 0 0% 90%;
    --popover: 0 0% 10%;
    --popover-foreground: 0 0% 90%;
    --primary: 0 0% 100%;
    --primary-foreground: 0 0% 6%;
    --secondary: 0 0% 16%;
    --secondary-foreground: 0 0% 90%;
    --accent: 0 0% 16%;
    --accent-foreground: 0 0% 90%;
    --muted: 0 0% 16%;
    --muted-foreground: 0 0% 65%;
    --destructive: 0 72% 50%;
    --destructive-foreground: 0 0% 100%;
    --border: 0 0% 25%;
    --input: 0 0% 25%;
    --ring: 0 0% 100%;
    --variant-error-bg: 0 60% 9%;
    --variant-error-border: 0 52% 28%;
    --variant-error-text: 0 68% 82%;
  }
}
```

Reference all colors via `hsl(var(--token))`. No hardcoded hex outside the accent. The accent is an accent only — never a large background.

Use **Lucide React** for icons (Lucide only).

---

## Reference snippets — use these

**`lib/amass.ts`** — singleton `AmassClient` with Bearer-auth, 429 retry/backoff (reads `Retry-After`), per-item-error helper (`mapLookupResult`), and `{data: ...}` envelope unwrap. Per Hard Rules #2, #3, #4, #5, #6, #11.

```ts
import "server-only";

type LookupResult = ReadonlyArray<{ amassIds?: string[]; error?: string }>;

interface AmassClientConfig {
  apiKey: string;
  baseUrl: string;
  maxRetries?: number;
  retryBaseMs?: number;
}

class AmassClient {
  constructor(private readonly cfg: AmassClientConfig) {}

  private async request<T>(method: "GET" | "POST", path: string, body?: unknown): Promise<T> {
    const url = `${this.cfg.baseUrl}${path}`;
    const headers: Record<string, string> = { Authorization: `Bearer ${this.cfg.apiKey}` };
    if (body !== undefined) headers["Content-Type"] = "application/json";
    const maxRetries = this.cfg.maxRetries ?? 3;
    const baseMs = this.cfg.retryBaseMs ?? 1000;

    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      const res = await fetch(url, {
        method, headers,
        body: body === undefined ? undefined : JSON.stringify(body),
      });
      if (res.status === 429 && attempt < maxRetries) {
        const retryAfter = Number(res.headers.get("Retry-After") ?? "1");
        await new Promise((r) => setTimeout(r, Math.max(retryAfter * 1000, baseMs * 2 ** attempt)));
        continue;
      }
      if (!res.ok) {
        const p = await res.json().catch(() => ({})) as { error?: { code?: string; message?: string } };
        throw new Error(`Amass ${method} ${path} → ${res.status} ${p.error?.code ?? "UNKNOWN"}: ${p.error?.message ?? res.statusText}`);
      }
      return res.json() as Promise<T>;
    }
    throw new Error(`Amass ${method} ${path} — retries exhausted`);
  }

  private async unwrapData<T>(p: Promise<{ data: T }>): Promise<T> {
    return (await p).data;
  }

  // Per-item-error helper per capability-map.md:121. Each result is either {amassIds: [...]} or {error: "..."}.
  // Per Hard Rule #11, the upstream-generated error string passes through verbatim — never paraphrased.
  static mapLookupResult<I, O>(
    inputs: I[],
    results: LookupResult,
    onSuccess: (input: I, amassIds: string[]) => O,
    onError: (input: I, message: string) => O,
  ): O[] {
    return inputs.map((input, i) => {
      const r = results[i];
      return r?.amassIds && r.amassIds.length > 0
        ? onSuccess(input, r.amassIds)
        : onError(input, r?.error ?? "no result");
    });
  }

  // Lookup: PMID/DOI → canonical AMBC_ (Hard Rule #6). The SR pre-screen uses the batch form
  // directly with chunks of ~100 PMIDs per the E-006 conservative items[]-ceiling heuristic.
  async batchLookupBiomed(items: ReadonlyArray<{ pmid?: string; doi?: string }>): Promise<LookupResult> {
    return this.request<LookupResult>("POST", "/api/v1/cores/biomedcore/records/lookup", { items });
  }
  async batchLookupTrial(items: ReadonlyArray<{ nctId: string }>): Promise<LookupResult> {
    return this.request<LookupResult>("POST", "/api/v1/cores/trialcore/records/lookup", { items });
  }
  async resolvePmid(pmid: string): Promise<string | null> {
    return (await this.batchLookupBiomed([{ pmid }]))[0]?.amassIds?.[0] ?? null;
  }
  async resolveDoi(doi: string): Promise<string | null> {
    return (await this.batchLookupBiomed([{ doi }]))[0]?.amassIds?.[0] ?? null;
  }
  async resolveNct(nctId: string): Promise<string | null> {
    return (await this.batchLookupTrial([{ nctId }]))[0]?.amassIds?.[0] ?? null;
  }

  // Per-paper GET — default includes []. The pre-screen post-filter reads the default-fields
  // (`isRetracted`, `journalQualityJufo`, `citationCount` per capability-map.md:163-169) without
  // an explicit include flag. Opt-in `include=referencesTrialCore` for the cross-core sub-workflow
  // when Step 1 dispatch enables it.
  async getBiomedRecord(
    amassId: string,
    includes: ReadonlyArray<"fulltext" | "authorsMetadata" | "meshIds" | "substanceIds" | "referencesTrialCore" | "references" | "citedBy"> = [],
  ): Promise<unknown /* bind to @amass/sdk BiomedRecord type once installed */> {
    const qs = includes.length > 0 ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.unwrapData(this.request<{ data: unknown }>("GET", `/api/v1/cores/biomedcore/records/${amassId}${qs}`));
  }

  async getTrialRecord(
    amassId: string,
    includes: ReadonlyArray<"detailedDescription" | "outcomes" | "referencesBiomedCore"> = [],
  ): Promise<unknown /* bind to @amass/sdk TrialRecord type once installed */> {
    const qs = includes.length > 0 ? `?${includes.map((i) => `include=${i}`).join("&")}` : "";
    return this.unwrapData(this.request<{ data: unknown }>("GET", `/api/v1/cores/trialcore/records/${amassId}${qs}`));
  }
}

let clientPromise: Promise<AmassClient> | null = null;

// Production Amass API base URL. Hardcoded by design — single production base, no per-user variance.
// To point at staging or a local proxy for development, edit this constant directly.
// Do NOT replace this with a process.env lookup — Lovable's secrets-UX will prompt the user for a
// value they don't have, which is unnecessary friction. Per Hard Rule #1.
const AMASS_API_BASE_URL = "https://api.amass.tech";

export function getAmassClient(): Promise<AmassClient> {
  return (clientPromise ??= Promise.resolve(new AmassClient({
    apiKey: process.env.AMASS_API_KEY!,
    baseUrl: AMASS_API_BASE_URL,
  })));
}

export { AmassClient };
```

**`app/api/prescreen-curated-ids/route.ts`** — Next.js App Router POST handler. Takes a JSON body `{pmids: string[], thresholds: {min_jufo, allow_retracted, min_citation_count}, include_referencesTrialCore?: boolean}`, chunks PMIDs into batches of 100 for `batchLookupBiomed`, runs the batch lookups concurrently, fans out per-paper GETs on the resolved `AMBC_` set wrapped in `withIdleTimeout(gen, 180_000)` per Hard Rule #7, post-filters client-side per Hard Rule #10, returns the filtered set + the audit-trail rows + lookup/walk error counts in one consolidated JSON response. Non-streaming for v0.1.

```ts
import { AmassClient, getAmassClient } from "@/lib/amass";

interface Thresholds {
  min_jufo: number;          // 0-3; JuFo tier post-filter threshold (default 2)
  allow_retracted: boolean;  // false = drop retracted; true = keep
  min_citation_count: number; // post-filter on returned citationCount (default 5)
}

interface PreScreenRequest {
  pmids: string[];
  thresholds: Thresholds;
  include_referencesTrialCore?: boolean; // opt-in cross-core sub-workflow
}

interface AuditRow {
  pmid: string;
  accessed_at: string;
  AMBC_id: string | null;
  isRetracted: boolean | null;
  journalQualityJufo: number | null;
  citationCount: number | null;
  referencesTrialCore_AMTCs: string[] | null;  // null when opt-in not enabled; [] when enabled but empty
  lookup_error: string | null;                  // per Hard Rule #11 — verbatim upstream string
  included_in_prescreen: boolean;
  // Run-time thresholds — same value repeats per row so a downstream reviewer can reproduce the
  // inclusion/exclusion verdicts without needing to remember what the screener set. Per finding 3
  // of the v0.1.0 Lovable empirical test adjudication (notes/decisions.md 2026-05-22 ADR).
  threshold_min_jufo: number;
  threshold_min_citation_count: number;
  threshold_allow_retracted: boolean;
}

interface FilteredPaper {
  pmid: string;
  AMBC_id: string;
  title?: string;
  authors?: string[];
  journal?: string;
  publicationDate?: string;
  doi?: string;
  abstract?: string;
  isRetracted: boolean | null;
  journalQualityJufo: number | null;
  citationCount: number | null;
}

async function* withIdleTimeout<T>(gen: AsyncGenerator<T>, maxIdleMs = 180_000): AsyncGenerator<T> {
  while (true) {
    const timeout = new Promise<{ kind: "timeout" }>((r) => setTimeout(() => r({ kind: "timeout" }), maxIdleMs));
    const result = await Promise.race([
      gen.next().then((r) => ({ kind: "next" as const, r })),
      timeout,
    ]);
    if (result.kind === "timeout") {
      throw new Error(`Stream stalled for ${Math.round(maxIdleMs / 1000)}s — an Amass call likely hung.`);
    }
    if (result.r.done) return;
    yield result.r.value;
  }
}

// Per Hard Rule #10 — `isRetracted` is the server-side default field; `journalQualityJufo` + `citationCount`
// are returned-field client-side post-filters per the quality-gate-papers MCP-vs-HTTP gap. Decision lives here.
function passesCredibilityFilter(
  record: { isRetracted: boolean | null; journalQualityJufo: number | null; citationCount: number | null },
  t: Thresholds,
): boolean {
  if (record.isRetracted === true && !t.allow_retracted) return false;
  if ((record.journalQualityJufo ?? -1) < t.min_jufo) return false;
  if ((record.citationCount ?? 0) < t.min_citation_count) return false;
  return true;
}

function chunk<T>(arr: T[], size: number): T[][] {
  const out: T[][] = [];
  for (let i = 0; i < arr.length; i += size) out.push(arr.slice(i, i + size));
  return out;
}

export async function POST(req: Request) {
  try {
    const body = (await req.json()) as PreScreenRequest;
    const client = await getAmassClient();
    const accessed_at = new Date().toISOString();
    const includes = body.include_referencesTrialCore ? (["referencesTrialCore"] as const) : ([] as const);

    // Step 1 — chunk PMIDs into batches of ~100 (E-006 conservative items[]-ceiling per
    // unfair-advantages.md:118 quantification-not-yet-empirically-tested marker) and run lookups
    // concurrently. Per-item-error rows populate even when the batch returns HTTP 200.
    const pmidItems = body.pmids.map((pmid) => ({ pmid }));
    const lookupChunks = chunk(pmidItems, 100);
    const lookupResults = await Promise.all(
      lookupChunks.map((items) => client.batchLookupBiomed(items)),
    );

    const auditRows: AuditRow[] = [];
    const amassIdsToWalk: string[] = [];

    lookupChunks.forEach((items, chunkIdx) => {
      AmassClient.mapLookupResult(
        items,
        lookupResults[chunkIdx],
        ({ pmid }, amassIds) => {
          amassIdsToWalk.push(amassIds[0]);
          auditRows.push({
            pmid: pmid!,
            accessed_at,
            AMBC_id: amassIds[0],
            isRetracted: null,
            journalQualityJufo: null,
            citationCount: null,
            referencesTrialCore_AMTCs: body.include_referencesTrialCore ? [] : null,
            lookup_error: null,
            included_in_prescreen: false,
            threshold_min_jufo: body.thresholds.min_jufo,
            threshold_min_citation_count: body.thresholds.min_citation_count,
            threshold_allow_retracted: body.thresholds.allow_retracted,
          });
          return null;
        },
        ({ pmid }, msg) => {
          // Per Hard Rule #11 — upstream `error` string passes through verbatim.
          auditRows.push({
            pmid: pmid!,
            accessed_at,
            AMBC_id: null,
            isRetracted: null,
            journalQualityJufo: null,
            citationCount: null,
            referencesTrialCore_AMTCs: null,
            lookup_error: msg,
            included_in_prescreen: false,
            threshold_min_jufo: body.thresholds.min_jufo,
            threshold_min_citation_count: body.thresholds.min_citation_count,
            threshold_allow_retracted: body.thresholds.allow_retracted,
          });
          return null;
        },
      );
    });

    // Step 2 — fan out per-paper GETs on the resolved AMBC_ set. Wrapped in withIdleTimeout per Hard Rule #7.
    // Per Hard Rule #10: default-fields fetch is enough for the three trust signals (no include flag
    // mandatory; opt-in include=referencesTrialCore only when Step 1 dispatch enabled it).
    const filtered: FilteredPaper[] = [];

    async function* walkPapers(): AsyncGenerator<{ amassId: string; record: unknown | null; err: string | null }> {
      for (const amassId of amassIdsToWalk) {
        try {
          const record = await client.getBiomedRecord(amassId, includes);
          yield { amassId, record, err: null };
        } catch (e) {
          yield { amassId, record: null, err: e instanceof Error ? e.message : "walk failed" };
        }
      }
    }

    for await (const { amassId, record, err } of withIdleTimeout(walkPapers(), 180_000)) {
      const row = auditRows.find((r) => r.AMBC_id === amassId);
      if (!row) continue;
      if (err) {
        // Per Hard Rule #10/11 — walk-side failure goes to lookup_error column (single error column
        // in the v0.1 audit-trail schema; the regulatory-evidence-assembler precedent uses a 2-column
        // lookup/walk split, but the SR pre-screen audit-CSV schema collapses to one column for
        // Rayyan/Covidence ingestion-shape parity).
        row.lookup_error = err;
        continue;
      }
      const r = record as {
        title?: string; authors?: string[]; journal?: string; publicationDate?: string;
        doi?: string; abstract?: string;
        isRetracted?: boolean | null; journalQualityJufo?: number | null; citationCount?: number | null;
        referencesTrialCore?: string[];
      };
      row.isRetracted = r.isRetracted ?? null;
      row.journalQualityJufo = r.journalQualityJufo ?? null;
      row.citationCount = r.citationCount ?? null;
      if (body.include_referencesTrialCore) {
        row.referencesTrialCore_AMTCs = r.referencesTrialCore ?? [];
      }
      const credible = passesCredibilityFilter(
        { isRetracted: row.isRetracted, journalQualityJufo: row.journalQualityJufo, citationCount: row.citationCount },
        body.thresholds,
      );
      row.included_in_prescreen = credible;
      if (credible) {
        filtered.push({
          pmid: row.pmid,
          AMBC_id: amassId,
          title: r.title,
          authors: r.authors,
          journal: r.journal,
          publicationDate: r.publicationDate,
          doi: r.doi,
          abstract: r.abstract,
          isRetracted: row.isRetracted,
          journalQualityJufo: row.journalQualityJufo,
          citationCount: row.citationCount,
        });
      }
    }

    return Response.json({
      filtered,
      audit_trail: auditRows,
      errors: {
        lookup: auditRows.filter((r) => r.lookup_error).length,
      },
      summary: {
        input_count: body.pmids.length,
        resolved_count: auditRows.filter((r) => r.AMBC_id !== null).length,
        filtered_count: filtered.length,
        thresholds: body.thresholds,
      },
    });
  } catch (err) {
    return Response.json({ error: err instanceof Error ? err.message : "Pre-screen run failed" }, { status: 500 });
  }
}
```

**Client-side RIS emission.** The Rayyan/Covidence-importable RIS file is emitted client-side from the `filtered` array in the response. The RIS field mapping is `TY=JOUR` per record + `PMID` (mono) + `TI` (title) + `AU` (one per author) + `JO` (journal) + `JF` (full journal name) + `PY` (publication year — first 4 chars of `publicationDate`) + `DO` (DOI) + `AB` (abstract) + `ER` (end-of-record). The audit-trail CSV is emitted client-side from `audit_trail` with the 11-column shape `pmid, accessed_at, AMBC_id, isRetracted, journalQualityJufo, citationCount, lookup_error, included_in_prescreen, threshold_min_jufo, threshold_min_citation_count, threshold_allow_retracted` (`referencesTrialCore_AMTCs` appears as a 12th column when the Step 1 opt-in is enabled). The three trailing `threshold_*` columns repeat the same run-time threshold values per row — the redundancy is deliberate; a single CSV file is self-contained for reproducibility without needing the screener to remember what thresholds were in force at run time.

The two snippets above are the only `lib/<file>.ts` + `app/api/<route>/route.ts` mandated by this prototype. v0.1 returns the assembled response in-memory in a single consolidated JSON — there is no persistence module, no `lib/prescreen-store.ts`, no monthly re-audit cron, no Python CLI sidecar. RIS download / CSV download happens client-side from the response payload. If a downstream extension needs persistence (monthly re-audit history per JTBD-8, in-tool BiomedCore-search discovery per v1.0), the extension SKILL.md includes its own minimal stub per the template's policy — never an `import` from `@/lib/<x>` without a matching snippet.

---

## Required reading

- `https://api.amass.tech/docs/biomedcore` — BiomedCore endpoint reference (batch lookup, get-by-ID, include flags, default fields including `isRetracted` / `journalQualityJufo` / `citationCount`) `[TODO: confirm Amass docs URL]`
- `https://api.amass.tech/docs/auth` — auth setup (Bearer token, env-var conventions) `[TODO: confirm Amass docs URL]`
- `https://api.amass.tech/docs/rate-limit` — rate limit (60 req / 60 s, per user+org) + error semantics `[TODO: confirm Amass docs URL]`
- `https://api.amass.tech/docs/capability-summary` — capability-map summary `[TODO: confirm Amass docs URL]`
- `https://api.amass.tech/docs/cross-core` — paper↔trial cross-core edge semantics (`referencesTrialCore`) for the opt-in sub-workflow `[TODO: confirm Amass docs URL]`
- `04-opportunities/briefs/07-sr-pre-screen-skill.md` — source brief (PASS at round-2; ICP-2 PRISMA pre-screen credibility-filter; v0.1 batch-lookup-driven path; v1.0 BiomedCore-search-driven discovery `[lifecycle: post-fix-only]` per AP-1).
- `.claude/skills/quality-gate-papers/SKILL.md` — trust-filter post-filter discipline (the load-bearing rubric mandatory-cut compliance mechanism per Hard Rule #10); documents the MCP-vs-HTTP gap on `minCitationCount` as post-filter-only.
- `.claude/skills/worked-example-anchoring/SKILL.md` — worked-example anchoring discipline (skill #8); per-prototype tier + mode mapping in section (d). Prototype 3 = MEDIUM caveat-and-ship.
- `https://github.com/amass-technologies/public-amass-platform-starter-ts` — **official Amass Platform Starter** (Apache-2.0; Bun + Ink + Vercel AI SDK CLI REPL with BiomedCore + TrialCore tools wired up). Use for tool-definition shape reference + ground-truth env-var conventions (`AMASS_API_KEY` required; `AMASS_API_BASE_URL` overrides default `https://api.amass.tech`); use this kit's `lib/amass.ts` for web app API calls. Stack diverges (CLI vs Next.js+TS) — the starter is the engineer-facing complement.

---

## Hand-off

Build, lint, and typecheck must pass.

**First, ask about credentials.** The demo is scaffolded but `.env` is placeholder. Use `AskUserQuestion` if available; otherwise ask directly:

**Do you have Amass API credentials?**
- **Yes — I'll paste them in** _(default)_
- **No — I need to sign up** (~2 min at https://platform.amass.tech, includes free-tier credits `[TODO: confirm Amass trial-tier shape]`)
- **Skip — I'll wire them up later**

Branch on the answer:
- **Yes:** "Find your `AMASS_API_KEY` at https://platform.amass.tech in the API credentials section `[TODO: confirm credential-page path inside platform.amass.tech]` — copy and paste into `.env`." Then run `npm run dev` and show the summary below.
- **No:** "Sign up at https://platform.amass.tech, then come back." When ready, proceed as Yes.
- **Skip:** show the summary below but DO NOT run the dev server. Close with: "I scaffolded `.env` but did not verify end-to-end. Fill `.env` and run `npm run dev` once credentials are ready."

Then present the hand-off summary. The four verification steps bind to the GLP-1 RA in obesity illustrative SR scope `[identifier-verification caveat — example identifiers assembled without live web access; re-verify against PubMed before binding to a real SR workflow]`:

> **Your SR Pre-Screen Skill is ready.**
>
> Before first push (if your AI builder left scaffolding placeholders): remove any `data-lovable-blank-page-placeholder="REMOVE_THIS"` attributes, any `<img>` tags pointing at `cdn.gpteng.co` or similar builder-placeholder CDNs, and any leftover "your app will live here" boilerplate. These render invisibly but pollute the DOM and serve no purpose in production. Search the codebase for `REMOVE_THIS` and `gpteng` before committing.
>
> To run it: fill in `.env` and run `npm run dev`.
>
> Verify it works:
> - Click **Try sample** to load the GLP-1 RA in obesity illustrative PMID set (~10 representative PMIDs covering semaglutide / liraglutide / tirzepatide in obesity / weight-loss endpoints). Click **Run pre-screen** with the default thresholds (`min_jufo: 2`, `allow_retracted: false`, `min_citation_count: 5`). Confirm the audit-trail table renders ~10 rows with `isRetracted` + `journalQualityJufo` + `citationCount` populated from live Amass (not hardcoded) and `included_in_prescreen` set per the threshold conjunction.
> - Tighten the threshold to `min_jufo: 3` and re-run. Confirm any PMID whose returned `journalQualityJufo` is 2 (or null) flips to `included_in_prescreen=false` in the audit trail; the filtered-paper count drops; the dropped rows still appear in the audit trail with `included_in_prescreen=false` (audit-trail completeness preserved — never silent omission).
> - Click **Download RIS**. Confirm the file emits in Rayyan-importable format (`TY=JOUR` per record + `PMID` + `TI` + `AU` + `JO` + `JF` + `PY` + `DO` + `AB` + `ER`); open in Rayyan or Covidence to confirm the import succeeds end-to-end. Click **Download audit CSV**; confirm the 11-column shape `pmid, accessed_at, AMBC_id, isRetracted, journalQualityJufo, citationCount, lookup_error, included_in_prescreen, threshold_min_jufo, threshold_min_citation_count, threshold_allow_retracted` renders verbatim (12 columns when `include_referencesTrialCore` opt-in is enabled).
> - Paste an intentionally-invalid PMID `99999999` into the textarea alongside the GLP-1 RA sample; click **Run pre-screen**. Confirm the audit-trail row for `99999999` renders inline with `lookup_error` populated by the verbatim upstream string (e.g. `"Unknown PMID"`) per Hard Rule #11 and `included_in_prescreen=false`, without crashing the surface and without omitting the row.
>
> Want to extend it?
> - Add a monthly re-audit cron per JTBD-8 (`jobs-to-be-done.md:237-260`) — re-resolves each `AMBC_` via the original PMID and flags newly-retracted records between SR publication (t0) and SR update cycle (tN); surfaces in the SR update workflow as a "X papers in your published bibliography were retracted since the last cycle" alert. Anchors to canonical `AMBC_` IDs so the audit chain is reconstructible at any prior cycle.
> - Add v1.0 BiomedCore-search-driven in-tool discovery — exposes `GET /api/v1/cores/biomedcore/records?query=...&minJournalQualityJufo=2&isRetracted=false&minPublicationDate=...` as an in-tool seed-PMID-list generator for SR teams that want to bootstrap an upstream-search-equivalent corpus inside the tool. `[lifecycle: post-fix-only]` per AP-1 / `anti-pitches.md:37` until E-025/E-026 closes — ships only then.
> - Add the cross-core walk sub-workflow via `include=referencesTrialCore` on the per-paper GET — for SRs whose inclusion criteria intersect literature with a target trial set (e.g. SR scope "RCTs of semaglutide in obesity where the published paper references a registered NCT"). Renders the `referencesTrialCore_AMTCs` column in the audit CSV; empty arrays for review-article papers render as honest emptiness per E-007 at-scale-symmetry doc-silent caveat. **If your scaffold trimmed `lib/amass.ts` to only the methods this v0.1 actually calls** (`batchLookupBiomed` + `getBiomedRecord` + `mapLookupResult`), you'll need to re-add `getTrialRecord` and optionally `batchLookupTrial` / `resolvePmid` / `resolveDoi` / `resolveNct` helpers to walk papers→trials and back. The full method-set is documented in `06-prototype-kit/amass-skill-template.md` lines 305-360.
> - I'm done — just show me the summary
>
> Have questions? Join the Amass Developer Community on Discord: https://discord.com/invite/sEGaBHMhWa

---

## Provenance

`[per Workstream C Block 3c — Prototype 3 SKILL.md (amass-sr-pre-screen, brief 07, MEDIUM caveat-and-ship) drafted against Corti-shape template 94f1680 2026-05-21]`

Inputs: `06-prototype-kit/amass-skill-template.md` (Corti-comparable canonical template at commit `94f1680`); `06-prototype-kit/README.md` (kit framing at commit `e331e2a`); `06-prototype-kit/amass-regulatory-evidence-assembler/SKILL.md` (Prototype 1 precedent at commit `c5961be` — `lib/amass.ts` shape + Hard Rules 1-8 inheritance verbatim); `06-prototype-kit/amass-pipeline-monitor/SKILL.md` (Prototype 2 precedent at commit `0627e40` — cross-prototype consistency anchor); `04-opportunities/briefs/07-sr-pre-screen-skill.md` (brief PASS at round-2 narrow verification 2026-05-20; v0.1 batch-lookup-driven path; v1.0 BiomedCore-search-driven discovery `[lifecycle: post-fix-only]` per AP-1); `.claude/skills/worked-example-anchoring/SKILL.md` (skill #8 at commit `d3ac0c3`; Prototype 3 = MEDIUM caveat-and-ship per section (d) line 86); `.claude/skills/quality-gate-papers/SKILL.md` (MCP-vs-HTTP gap on `minCitationCount` as post-filter-only — load-bearing for Hard Rule #10); `01-capabilities/capability-map.md` (Amass API ground-truth — shared-auth, shared-rate-limit, shared-errors, BiomedCore §lookup + §get-by-ID + §default-fields with `isRetracted` / `journalQualityJufo` / `citationCount` at lines 163-169); `03-positioning/unique-values.md` (UV-4 trust-filter conjunction at non-commercial tier + UV-6 batch lookup with per-item-error + UV-1 cross-core spine + UV-7 canonical-ID stability).

MEDIUM caveat-and-ship tier per skill #8 section (d) line 86: workflow is screening-recommendation (analyst makes the final inclusion decision in Rayyan/Covidence downstream), not retraction-or-claim-assertion. No within-Prototype-3 verification dispatched; caveat marker `[identifier-verification caveat — example identifiers assembled without live web access; re-verify against PubMed before binding to a real SR workflow]` carried at 3 anchor sites (title intro paragraph + Try-sample tooltip + Hand-off verification preamble) per the worked-example-anchoring skill's MEDIUM caveat-marker variant. Display=claim defamation-adjacent guard structurally preserved by live-Amass-binding requirement at Hard Rule #12 (Lovable scaffolds against the live API; never against fabricated retraction fixtures).

`[v0.1.1 patch — 2026-05-22 — per the per-prototype publication patch pass discipline established in the 2026-05-22 Workstream C Block 4 close ADR. Bundles five findings from the v0.1.0 Lovable empirical test (FULL PASS verdict, Block 5.5): (1) stack-flexibility rewrite — Lovable defaults to TanStack Start; Next.js App Router supported as alternative; lib/amass.ts pattern stack-agnostic; (2) AMASS_API_BASE_URL friction removal — hardcoded as constant in lib/amass.ts; removed from Hard Rule #1 env block; (3) audit-CSV threshold columns — 8→11 base columns (added threshold_min_jufo + threshold_min_citation_count + threshold_allow_retracted) so reviewers can reproduce inclusion verdicts from a single self-contained CSV; (5) Hand-off placeholder cleanup note — pre-push instruction to remove Lovable scaffolding REMOVE_THIS / gpteng placeholder artifacts; (6) Want to extend it method-set trimming note — if scaffold trimmed lib/amass.ts to P3-only methods, need to re-add getTrialRecord + batchLookupTrial + resolvePmid/resolveDoi/resolveNct for cross-core sub-workflow. Finding 4 (threshold-vs-result mismatch on PMID 37532962 with JuFo=1) closed as non-issue — user confirmed they set min_jufo=1 at run time; filter logic correct. Per Block 5.5 adjudication on top of HEAD 7e55697.]`
