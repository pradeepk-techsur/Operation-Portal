# Build instructions — Contract Transparency Portal

Instructions for an LLM asked to build (or rebuild) this application. Written from the full design session; every requirement below was stated or corrected by the customer, and the pitfalls section records defects that actually occurred.

---

## 1. Objective

Build a single-page, customer-facing portal that gives government stakeholders visibility into contract execution across all call orders under a BPA, and gives Project Managers the place where they keep that information current.

Deliver it as one Design Component (`Contract Transparency Portal.dc.html`). Inline styles only. No stylesheets, no CSS classes.

Start simple. The customer's opening ask was literally "something very simple: list of call orders (name, period of performance, no of people, funded, actual spend); clicking the call order name opens a details page." Everything else was added in later rounds. Build in that order — a working register first, then depth.

## 2. Source data — read it, do not invent it

The program's real artifacts are the source of record. Parse them before designing:

| Artifact | Supplies |
| --- | --- |
| `Project Financial Analysis Next Gen.xlsx` → sheet **AO EAC Table** | The authoritative call order list: identifier, period of performance, funds obligated, expended to date, remaining, estimate at completion, over/under |
| Same workbook → per-call tabs (`AO Call 2.3`, `AO Call 4.3`, …) | Funded ceiling, hours forecast by resource and month |
| `Call Order Staffing.xlsx` → sheet **Resource by Call Order** | Roster by name, LCAT, rate; contracted BPA labor categories with FTE, hours, quoted rate; vacancies as named rows |
| `BPA Labor Rate Pricing Sheet.xlsx` | Authorized labor categories and ceiling rates for validation |
| Monthly Status Report (`.docx`) | Per-call-order funding statement, activities completed and planned, employee listing, requested travel, high-priority issues and risks, GFE log, billets, new contractors, departures, glossary |
| Weekly touchpoint agenda (`.docx`) | Per-call-order weekly items, current vacancy list, action items, PIV and onboarding status |

Parsing notes that cost time in this session:

- `.xlsx` / `.docx` are zip archives. Read the central directory, inflate entries with `DecompressionStream('deflate-raw')`, then parse `xl/worksheets/sheetN.xml` + `xl/sharedStrings.xml`, or `word/document.xml`.
- Map sheet names to files through `xl/workbook.xml` + `xl/_rels/workbook.xml.rels`; sheet order is not file order. The financial workbook here had 67 sheets.
- **Extract every call order section from the MSR, not a sample.** The first pass pulled only two sections and the customer's response was "the monthly status report is not parsed properly in the UI." The June 2026 report contained seven.
- Keep the customer's wording verbatim in narrative fields. Do not summarize, tighten, or rewrite activity text, risks, or issues.
- Where a spreadsheet row is ambiguous (unlabeled LCAT, blank quoted rate), omit it and say so — do not guess a value into the UI.

## 3. Information architecture

Two top-level pages, plus a call order detail view:

```
Masthead: product name + mono contract line (agency · vehicle · contract number)
Nav: [Call Orders] [Monthly Status Reports]        View as: (Customer | Project Manager)

Call Orders  → register table → call order detail
                                  ├── Option periods selector (when >1 funded period)
                                  └── Tabs: Financials | People | Weekly Status Reports
Monthly Status Reports → deliverable log → section reader (rail of call orders)
```

**The monthly status report is not a call order artifact.** It is one BPA-level contractual deliverable covering every call order. Do not attach it to a call order's tabs; give it its own top-level page. Weekly status reports are the opposite — always per call order.

## 4. Screens and behavior

### 4.1 Call order register

- One row **per call order**, not per funded period. Option periods roll up: show the current period's figures, and label the row with the call order group, the shown period, and how many periods exist (e.g. `Call 013 · Call 13.1 · 2 periods`).
- Determine the current period by date containment against today: a period whose end date has passed is prior, one that has not started is upcoming. A call order whose base period ended opens on its option period.
- Columns: call order (name + identifier line), period of performance, people, funded, actual spend (with burn bar and percentage), last updated.
- Portfolio totals in the header: obligated, expended. Summary line: `N call orders · M funded periods · P personnel assigned · V open positions`. Keep those counts consistent with what the table shows — a register of 8 rows must not say "10 call orders".
- Clicking the row opens the detail view on Financials. Clicking the **People number** opens the detail view on the People tab (stop propagation so the row handler does not override the tab).
- Every column sortable, both directions.

### 4.2 Financials tab

- Funding summary: funds obligated, funds expended to date*, funds remaining, estimate at completion, over/under, percent expended. Values to the cent. Footnote: expenditures lag one invoice cycle.
- Contracted labor categories: BPA labor category, authorized FTE, hours, quoted rate, extended value, with a total contracted value row.
- **Tables only. No charts.** The customer removed the monthly-expenditure bar chart explicitly; it became a table of month, expenditures, cumulative, percent of funded.
- Percent expended is colored by the burn thresholds (below).

### 4.3 People tab

- Tiles: contracted FTE, assigned, vacant, departed, labor categories.
- Roster: name, labor category, rate, status. Vacancies are roster rows carrying their LCAT and rate, colored with the flag hue. Departed staff are greyed.
- **Assigned headcount excludes both vacancies and departures.** A status of "Offboarded" or "No longer available" means the person is not assigned; otherwise assigned can exceed contracted FTE, which is wrong and was caught in review.
- Drill-down panel **By labor category**: each LCAT with contracted FTE, filled count, and bill rate, expanded to the resources in it with individual rates and status. Categories with nobody in them say so rather than rendering empty.
- Flag where assigned resources in a category exceed contracted FTE (the source data had 5 Systems Integration Engineers against 3 contracted).

### 4.4 Weekly status reports tab

- Per call order. A PM can **author** a report in the app — week ending, submitted by, accomplishments, planned activities, risks, issues, customer actions and decisions, one item per line — or **upload** a document.
- Submitted reports are retained in a log (period, file, submitter, status) and readable in place, grouped by section. Uploaded documents link to the source file; a freshly uploaded file shows a "sections extracted once processed" state.
- Where a program-wide touchpoint covers several call orders, show only the items belonging to the call order being viewed.

### 4.5 Monthly status report page

- Log of deliverables: reporting period, file, submitted by, due date, status (Draft / Submitted / Accepted).
- Three ways in: **New report** (blank draft for a period, filled section by section), **Draft from portal data** (funding pulled from the portal, activities/risks/issues assembled from the weekly reports PMs submitted in that period), **Upload file**.
- Reader: left rail of every call order; selecting one shows its section — funding information, activities completed, activities planned, high-priority risks, high-priority issues, requested travel, employee listing. Call orders with no section in that report are dimmed, not hidden.
- PM authoring: **Edit section** / **Add section for this call order** opens a form for that call order — five funding fields (pre-filled from the section or from portal financials), activities completed and planned (one per line; a leading `Workstream: text` becomes a bold heading), risks, issues, travel.

### 4.6 Roles

- Header switch: **View as Customer | Project Manager**.
- **The customer has no ability to create, edit, upload, or delete anything, anywhere.** Every create/upload/edit affordance is hidden in customer view — upload call order, both report uploaders, the authoring forms, roster editing, financial editing. Enforce server-side too, not by hiding controls.
- PM view turns the same screens into editors: expended-to-date field on Financials; add/remove person and change status on People; report authoring.

### 4.7 Data currency

- Financial and staffing records each carry a last-updated stamp, shown wherever the record appears. Any save re-stamps it.
- The register shows the oldest stamp per call order and flags anything past the refresh interval (staffing weekly, financials monthly; make the interval configurable).
- Attribute every change to a user and retain an audit history.

## 5. Design rules

Neutral, dense, government reporting instrument. Full detail lives in the project's design system (`readme.md`, `tokens/`); the essentials:

- **Type**: IBM Plex Sans for readable text, IBM Plex Mono for every label, identifier, date stamp and status token. Body 13px/1.55. Mono eyebrows 10px uppercase, 0.08em tracking. Page titles 26px/600 at −0.02em. Weights 400/500/600 only. Tabular numerals on every figure.
- **Color**: warm-neutral grey ramp on four white-ish surface steps; one accent `oklch(0.52 0.09 245)` for links, active state, primary buttons and sorted columns; `oklch(0.62 0.14 45)` flags vacancies and stale records. At most two background colors per screen.
- **Burn thresholds** (fixed policy, applied to the bar, the register percentage, and percent expended): below 75% accent, 75–85% `oklch(0.72 0.15 78)`, 86%+ `oklch(0.55 0.2 27)`.
- **Structure**: 1px hairlines only. No shadows, no gradients, no imagery, no icons beyond Unicode arrows and dashes, no emoji, no animation. Radii 2px and 3px.
- **Density**: table cells 12px × 20px, column gap 16px, card padding 18px/22px, page inset 32px, max width 1240px. The spacing scale includes odd values on purpose — copy them.
- **Copy**: sentence case; contract vocabulary used exactly (obligated, expended, estimate at completion, period of performance, labor category, FTE); no second person; empty states explain the reason in a full sentence; buttons are verb + object.

## 6. Tweakability

No host-level tweak props. The customer removed sort-order, spend-bar, and default-tab props with "I dont want sortby, showspendbars, defaulttab. All tables should have all columns sortable." Behavior that matters belongs in the UI, not in a settings panel.

## 7. Pitfalls that actually occurred — check for each

1. **Departed staff counted as assigned.** Filtering only names starting with `VACANT` left offboarded people in the headcount, so assigned exceeded contracted FTE. Exclude departures explicitly.
2. **Fuzzy LCAT matching mis-filed resources.** A prefix match on a truncated normalized name collapsed "Business Analyst – Mid" into "– Senior", moving a $97.97 vacancy into the wrong category. Match normalized names for exact equality first; only then fall back to a full-string prefix match.
3. **Status dropdown discarded free-text notes.** A controlled `<select>` reported "Assigned" for every row whose stored status was a note ("Moving to Business Architect", "Offboarded 8/28"), and touching it overwrote real data. Prepend the stored value as an option when it is outside the vocabulary.
4. **Shape mismatch between parsed and authored content.** Parsed MSR activities were `[title, text]` arrays while the draft builder emitted `{title, text}` objects, so drafted sections rendered blank rows. Normalize on one shape.
5. **Dead "open source document" links** on portal-drafted and uploaded records. Only render the link when a real file backs it.
6. **Rollup / summary drift.** After grouping option periods, the summary line still counted funded periods as call orders.
7. **A helper deleted during a rewrite.** A full-file rewrite dropped a style helper that a later feature still called, and the whole page rendered blank below the masthead. After any rewrite, check that every helper referenced in render is still defined.
8. **Sorting broke row-level actions.** Roster edits addressed rows by display index; after sorting, edits hit the wrong person. Carry the original index on each row.

## 8. Definition of done

- The register, both option-period behavior and burn coloring, matches Section 4.1.
- All seven MSR sections from the source document are readable, with funding, activities, risks, issues, travel and employee listing.
- A PM can author a weekly report and a monthly section, and upload files, in the app.
- Customer view shows no authoring affordance anywhere.
- Every table sorts on every column, in both directions, without corrupting row actions.
- Assigned never exceeds contracted FTE; no resource appears under a labor category it does not hold.
