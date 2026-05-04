# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project shape

Single-file static web app: `index.html` is the entire project (~3900 lines, ~480KB). Vanilla JS, no framework, no build, no tests, no package manager. The bulk of the file size is two inline JSON data tables: `CP_CERTS` (the Cisco cert catalog) and `CP_BB` (the Black Belt training catalog). One external dep loaded from CDN: SheetJS (`xlsx.full.min.js`) for parsing uploaded Excel files.

## Deployment

GitHub Pages: **https://curt847.github.io/pvi/** — `main` branch, root. Push to `main` rebuilds and publishes. Remote: `git@github.com:curt847/pvi.git` over SSH (public repo).

## What it does (two apps in one page)

The same page has two distinct features that share a header but otherwise have separate state and rendering paths:

1. **PVI Calculator.** User uploads a "Partner Assessment DPV" `.xlsx`. The app parses rows into `allRows`, computes per-portfolio PVI scores per month, renders ribbons / milestones / a 12-month trend chart / per-portfolio panels with measurement detail, and supports a **what-if** mode where the user can override applied indexes per measurement and see live recomputed scores without rebuilding the DOM. PVI math is in the section starting around line 982 — `calcPVI(port, mon, wi)` is `sum(catWeight × measWeight × (whatIf ? overrideIdx : appliedIdx))` per row.

2. **Cert Planner.** User uploads a personnel roster (and optionally a Black Belt completions file), and the app produces a per-person view of cert/BB attainment vs. PVI gaps. The core feature is **AI-generated training plans**: free-form (`cpRunFreeformPlan`) and manual (`cpRunManualPlan`), both calling the Anthropic API directly from the browser. The user can enter the app and skip directly to this screen via "Skip — go to Cert Planner" (`skipToCertPlanner()`), which bypasses the DPV requirement.

The two apps' state is independent. PVI uses top-level vars (`allRows`, `months`, `selectedMonth`, `whatIfMap`, `partnerName`, `beId`, `shirtSize`, `chartFilter`). Cert Planner uses its own `cp*` globals (and constants `CP_CERTS`, `CP_BB`).

## Anthropic API integration

The Cert Planner calls `https://api.anthropic.com/v1/messages` **directly from the browser** with the user's key, using the `anthropic-dangerous-direct-browser-access: true` header to bypass the SDK's same-origin guard. The key is read from a UI input (`#cp-api-key`) and persisted to `localStorage['pvi_api_key']`. There is no server-side proxy.

**This is intentional and not a bug.** PVI is personal-use only — Curt enters his own Anthropic key. There is no plan to roll it out to Cisco partners under the current funding model (no corporate Anthropic key available). Don't re-propose adding a server-side proxy unless the user mentions a corporate key or specific partners they want to onboard. If/when that changes, the rollwatch proxy pattern (`PropertiesService` + auth + per-user rate limit + daily total cap) is the template.

Models in use:
- `claude-haiku-4-5-20251001` — only for the "Test Key" round-trip (cheap)
- `claude-sonnet-4-6` — actual plan generation (three call sites: `~2796`, `~3571`, `~3845`)

If you change models or system prompts, change them at all the call sites.

## Threshold tables are authoritative external data

The big top-of-script tables — `BB_THRESH`, `CAREER_THRESH`, `ACV_THRESH`/`TCV_THRESH`, `PREM_THRESH`, `OTRR_THRESH` (lines ~759–834) — are transcribed from **Cisco PVI Planner v8.6 Metric Formulas sheet**. Don't tweak the numbers casually; they're the source of truth for partner scoring. If Cisco publishes a new version of the sheet, update by replacing these blocks wholesale and bumping the comment at line 755.

`PORTFOLIOS` (line 838) defines the canonical portfolio set and order — adding/removing one cascades through every render path. `MILESTONES` (line 836) defines the tier breakpoints (0/1/5/7.5).

## What-if rebuild discipline

`handleWIF()` (line ~1340) intentionally **does not** call `renderAll()` — it does an in-place DOM update on the affected row(s) plus the ribbon/peak card to avoid layout shift while the user is typing. If you add a new what-if-affected UI element, hook it into the in-place updater, not the full render. The full `renderAll()` is for data-load and tab-switch.

## Plan post-processing pipeline

The AI returns a markdown plan, but the raw output isn't shown directly. It runs through several passes (in order):

1. `cpReassignByRole` / `cpReassignToQualified` — fix assignee/role mismatches
2. `cpEnforceStagePrerequisites` — prevent Stage 2/3 trainings from appearing without their prerequisites
3. `cpFilterByTargets` — drop rows outside the user's target portfolios
4. `cpTopUpBBPlan` — if BB targets aren't met, scan `CP_BB` for the best next item (cross-arch first, shortest duration, role-eligible, prereqs satisfied) and append until each target hits or no eligible items remain
5. `cpBuildPortfolioSummary` — render the totals table
6. `cpFormatPlan` — final markdown → HTML

Each pass operates on plain text and is order-dependent. The top-up pass exists because the model often produces a too-short plan; trust deterministic top-up over re-prompting.

## Things worth knowing

- API key is **never embedded in source** (verified). User enters it in the header input; the test button does a 5-token Haiku ping to validate.
- The "Skip — go to Cert Planner" path hides several DPV-dependent UI elements via inline `style.display='none'`. `initUI()` un-hides them when a real DPV is later loaded.
- Excel parsing tolerates messy column headers via `findIndex(h => h.includes(name))` lookups in `parseData()` — column order doesn't matter, but renamed/missing columns silently produce zeros.
- `selectedMonth` defaults to the most recent month found in the file (`months` is sorted descending by integer prefix). If the file's month strings don't start with parsable integers, sort breaks subtly.
- Print/PDF flows (`ovPrint`, `cpPrintPlan`, `cpPrintRoster`, `cpPrintBBRoster`) open new windows with hand-built HTML — they don't reuse the live DOM.
