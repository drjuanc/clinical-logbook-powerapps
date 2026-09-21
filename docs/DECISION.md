# DECISION.md — Electronic Clinical Logbook

**Project:** Student clinical procedures logbook
**Platform:** Microsoft 365 tenant (Power Apps canvas + SharePoint + Power Automate)
**Status:** Design settled · build in progress — data foundation and content loaded, identity flow and theme built (see PLAN.md for phase state)
**Last updated:** 12 August 2026
**Open blockers:** §3.1 items 1 and 8 — `LogEntries_DEV` has no `SupervisorName` column; `YearLevel` / `Year` choice text may not match.

---

## 1. Purpose

Replace the paper clinical logbook with an electronic system that tracks procedures performed by students across Years 1, 2 and 3. Each year has a distinct list of required procedures, each with a target count per student. The system must produce defensible evidence for HPCSA accreditation (who did what, where, verified by whom).

---

## 2. Settled decisions

### 2.1 Front-end — Canvas Power App
**Decision:** Build a canvas app, not a Lists form or Forms + Flow.
**Rationale:** Only a canvas app can filter procedures by the student's year, show live progress (logged vs required), capture a signature and GPS in one flow, and run on a phone at the bedside. Uses standard connectors only — no premium licensing.

### 2.2 Student identity — existing Students list, Person column
**Decision:** Reuse the existing **Students** SharePoint list. Identify the logged-in student via the **Student** Person column against `User().Email`. No new student list, no separate authentication.
**Rationale:** Single source of truth; inherits the tenant identity model already in use. `Dereg`/`Dereg_Date`/`Dereg_reason` already provide an audit-friendly active/inactive state, replacing any need for an `Active` flag.
**Note:** The **Title** column (student card barcode) is **not** used in this app — it belongs to a separate app. Identity is via the Person column only.

### 2.3 Supervisors — externals cannot authenticate
**Decision:** Supervisors are frequently external (rotating nurses, sessional doctors) with no tenant account. Verification therefore does **not** rely on login, PIN, or email.
**Rationale considered and rejected:**
- *AAD guest accounts for every external* — administrative churn, unmanageable at peripheral sites.
- *Supervisor PIN* — leaks in small sites; not viable for externals.
- *Email verification link* — ruled out; supervisors cannot be relied upon to have/check email.

### 2.4 Verification model — signature + identity + geolocation (FINAL)
**Decision:** Each entry is countersigned at the bedside by capturing:
- Supervisor **name + surname** (from registry, or typed ad hoc for first encounters)
- Supervisor **designation** (Dr / Professional Nurse / Clinical Associate, etc.)
- **Signature** via the Pen Input control
- **GPS coordinates** + derived human-readable site match
- **Timestamp** (`Now()` at submission, plus server-side Created time as a harder-to-spoof backup)

**Rationale:** Mirrors the paper logbook's wet signature — a familiar, HPCSA-legible artefact — but adds location and timestamp that paper cannot. Deterrence against forgery comes from **monthly random audit sampling** per site, announced to students.

### 2.5 Geolocation handling
**Decision:** Capture location **only at the moment of logging** (never continuous tracking). Compute distance to the student's registered hospital at capture time and store a derived, readable `SiteMatch` (e.g. "St Barnabas (140 m)"). GPS failure must **flag** the entry (`LocationStatus = Unavailable`), never silently store zeros.
**Rationale:** POPIA — minimise and justify processing of student location data; cover in programme policy + student consent. Readable site match makes reports and anomaly-spotting trivial.

### 2.6 Data integrity — students never write directly
**Decision:** Students have **no direct edit access** to the LogEntries list. All writes are routed through a Power Automate flow running under a **service account**.
**Rationale:** Prevents students bypassing the app to doctor their records. The flow also handles the Pen Input image conversion (Pen Input cannot patch straight into a SharePoint image column), so this costs nothing extra.

### 2.7 Mobile-first, responsive layout (LOCKED)
**Decision:** The app is **mobile-first** — designed for a phone held in the hand at the bedside, responsive enough not to break on a tablet, but never optimised for desktop. Layout is built with **responsive containers** (horizontal + vertical), not fixed X/Y coordinates.
**Rules:**
- Every screen uses a root **vertical container** (`FillPortions`-based) so content reflows to any screen height; rows within use **horizontal containers**. Avoid absolute positioning except for genuinely floating elements (e.g. a fixed submit bar).
- Design against a narrow portrait viewport first; let containers stretch up, never assume width.
- Tap targets sized for thumbs; primary action (Submit) always reachable one-handed near the bottom.
- No horizontal scrolling. Single-column flows on phone.
**Rationale:** Students log on personal phones of varying sizes at peripheral sites. Containers make one build adapt to all devices and orientations without duplicate screens.

### 2.8 Performance on low-speed networks (LOCKED)
**Decision:** The app must be **light and fast on poor rural-Africa connectivity**. Optimise aggressively for first load and minimise per-action network round-trips.
**Rules:**
- **Preload + cache slow-changing lists into collections** once at startup: **Procedures**, **Supervisors**, **Hospitals** change rarely, so pull them once (delegably) into collections and read from the collection thereafter — not the live list on every keystroke.
- Wrap independent startup loads in **`Concurrent()`** so they fetch in parallel, not serially.
- Keep **`OnStart`/`App.StartScreen` work minimal**; defer non-critical loads until after the first screen paints, or load on-demand per screen.
- Only fetch the **current student's own** data live; never pull the whole LogEntries list to the device — filter server-side (delegable) to `Student.Email = User().Email`.
- **Never re-fetch signature images** to the device for browsing; they live server-side and are only needed for audit/coordinator views, loaded on demand.
- Keep control count per screen low; fewer controls = faster render on low-end phones.
- Prefer **collections and variables** over repeated `LookUp`/`Filter` calls that each hit the network.
- Add a **manual "refresh reference data"** action (re-runs the collection load) so cached lists can be updated without reinstalling — students refresh only when told procedures/supervisors changed.
**Rationale:** On a slow or intermittent connection, every avoided round-trip is the difference between a usable and an abandoned app. Static reference data cached locally means logging a procedure needs almost no bandwidth beyond the single write.

### 2.9 Theming — WSU brand tokens as named formulas (LOCKED)
**Decision:** The palette is derived from the official WSU logo and exposed as **named formulas** in `App.Properties.Formulas`. Screens reference tokens (`ThemePrimary`, `ThemeInk`, `ThemeSpaceM`) and **never** a raw hex value or pixel constant. Full palette, roles and contrast figures: **`docs/THEME.md`**.
**Rules:**
- Colours are sampled from `docs/design/WSU_new_logo.png` itself, not approximated. `#551516` maroon anchors the palette; the amber, sage and crimson chevrons supply the accent and semantic colours, so status colours are brand-native rather than generic web green and red.
- Every token is **contrast-checked against all three background tokens** before being added, and the worst-case ratio recorded. Targets: 4.5:1 normal text, 3:1 large text and non-text UI boundaries. **A token that has not been contrast-checked does not go in the palette.**
- **`ThemeAccent` (`#CF8029`) is fills and graphics only** — at 2.82:1 it fails AA for text and misses the 3:1 non-text threshold on the canvas background. `ThemeAccentDeep` is the text-safe amber.
- Type comes from the fonts Power Apps ships natively (`Open Sans`), so nothing is fetched over the network at launch.
- Tap targets are at least `ThemeTouchTarget` (48).
**Rationale:** One definition means a brand change is a single edit rather than a search across every screen, and it makes accessibility enforceable — contrast is verified once per token instead of re-argued per control. Sampling from the logo rather than eyeballing keeps the app visibly the university's.

**Accessibility checker note:** the built-in checker reports ~33 findings against static labels, prescribing `TabIndex = 0`. This is **deliberately not applied** — it would place every static caption in the keyboard tab order, which is worse for keyboard and switch users. Static text carries `TabIndex: =-1`. Real interactive controls (buttons, inputs, Pen Input) get `TabIndex: =0` **and** a visible focus border, and their findings must be taken seriously. Reasoning in `docs/THEME.md` §6.

---

## 3. Data model (settled)

> **Status as at 12 Aug 2026:** all five lists exist in the DEV environment and are connected to the canvas app. The tables below record the **as-built** schema — internal names verified against the live lists via the Canvas Authoring MCP. Where the as-built state differs from the locked design, it is marked and carried into §3.1.

| Role | Actual list (DEV) | Status | Key columns (internal names) |
|---|---|---|---|
| **Students** | `StudentList_DEV` | Exists — no change | `Student` (Person, identity for `User().Email` match), `Number` (student number), `Name`, `Year`, `Hospital` (lookup), `Pic`, `Dereg*` |
| **Hospitals / Sites** | `TSites_DEV` | Exists — geofence columns **present** | `Title`, `acronyms`, `Latitude`, `Longitude`, `Radious` (⚠ misspelled — metres), `Acitve` (⚠ misspelled — Boolean), `Location` (text), `Coordinator` / `Admin` / `Lecturer` (Person), `Picture`, `Notes` |
| **Procedures** | `Procedure_DEV` | Exists | `Title`, `YearLevel` (Choice), `RequiredCount` (Number), `Category` (Choice), `Module` (Choice — not in original design), `Active` (Boolean), ⚠ `RequiredCount0` (stray duplicate — delete) |
| **Supervisors** | `Supervisor_DEV` | Exists — matches design | `Title` (full name), `Designation` (Choice), `CouncilNumber` (Text), `Site` (Lookup→TSites), `Status` (Choice: Registered/Pending) |
| **LogEntries** | `LogEntries_DEV` | Exists — flow-write only | See detailed schema below |

**DEV/PROD environments** are managed via environment variables. All list names in this project have a `_DEV` suffix in development and drop the suffix in production; solution references use variables, not hardcoded names, so cutover requires no rebuilding of Lookups or formulas.

**Internal names are permanent.** SharePoint fixes a column's internal name at creation; renaming afterwards changes only the display name. Power Fx binds to the **internal** name. The misspellings above (`Radious`, `Acitve`) are therefore load-bearing — write formulas against them as spelled, and do not "correct" them in code.

### LogEntries — detailed schema (as built)

Internal names as verified in `LogEntries_DEV`. Use these exact strings in Power Fx and in the write flow.

| # | Internal name | Display | Type | Notes |
|---|---|---|---|---|
| 1 | `Title` | Title | Text | Computed by the write flow, e.g. `"Ndlovu S · Suturing · 11 Aug 2026"` — makes SharePoint default views readable. Not the identifier. |
| 2 | `Student` | Student | **Lookup → StudentList_DEV** | Primary display: `Number` (student number). Referential integrity enforced. Written by the flow using `varStudent.ID`. |
| 2a | `Student_x003a__x0020_Name` | Student: Name | Lookup projection | Read-only projection of the student's `Name`, for display in SharePoint views. Not written. |
| 3 | `Procedure` | Procedure | Lookup → `Procedure_DEV` | Referential integrity to Procedures. |
| 4 | `Supervisionlevel` | Supervision level | Choice | Observed / Assisted / Performed under supervision / Performed independently. ⚠ Internal name is lower-case `l` — not `SupervisionLevel`. |
| 5 | `Supervisor` | Supervisor | Lookup → `Supervisor_DEV` | The design's `SupervisorRef`. Blank if ad-hoc (supervisor not yet in registry). |
| 5a | `Supervisor_x003a__x0020_Title` | Supervisor: Title | Lookup projection | ⚠ **Not a substitute for `SupervisorName`** — this is a live projection, not a copy. See §3.1 item 1. |
| 6 | — | — | — | 🔴 **`SupervisorName` (Text) is MISSING.** Required by the audit model. See §3.1 item 1. |
| 7 | `SupervisorDesignation` | SupervisorDesignation | ⚠ **Text** (design: Choice) | Should be a Choice of: Medical Doctor / Medical Intern / Professional Nurse / Enrolled Nurse / Clinical Associate / Other Health Professional. See §3.1 item 2. |
| 8 | `Signature` | Signature | **Image column** | The design's `SignatureImage`. Written by the flow, which strips the `data:image/png;base64,` prefix from Pen Input, decodes to binary, and uploads. Renders as thumbnail in SharePoint views. |
| 9 | `Latitude` | Latitude | Number | Raw GPS at moment of logging. Stored for audit; never displayed raw to students. |
| 10 | `Longitude` | Longitude | Number | As above. |
| 11 | `Sitematch` | SiteMatch | Text | Human-readable: `"St Barnabas (140 m)"` or `"No site within radius"`. Computed by the app via haversine against the student's site, passed to the flow. ⚠ Internal name is lower-case `m`. |
| 12 | `LocationStatus` | LocationStatus | Choice | Captured / Unavailable. Blocks silent `(0,0)` storage; drives the "Flagged" status downstream. |
| 13 | `EntryStatus` | EntryStatus | Choice | Valid / Flagged / Audit-verified. Defaults to Valid on write; auto-set to Flagged if `LocationStatus = Unavailable`; set to Audit-verified by the monthly sampling flow. |
| 14 | `EntryDate` | EntryDate | Date only | When the procedure was actually performed. Student-selectable. **Defaults to `Today()`, hard-blocked to the range `Today()-7 ≤ date ≤ Today()`** — no future, no backdating beyond 7 days. Distinct from SharePoint auto-`Created` timestamp; a gap between them is an audit signal. |
| 15 | `Notes` | Notes | Multiline text | Optional case context. **No patient identifiers** — enforced by app-side helper text + POPIA consent, not by SharePoint. |
| ? | `Performace` | Performace | Number | ⚠ Not in the design. Purpose unconfirmed; name appears to be a misspelling of "Performance". See §3.1 item 3. |

Plus SharePoint's automatic system columns (do not add): `ID` (auto integer, the true identifier), `Created` (server timestamp — harder to spoof than a client-set date), `Author` / "Created By" (will always be the write flow's service account, not the student), `Modified`, `Editor` / "Modified By".

**Design rules embedded in this schema:**
- **Copied vs looked-up supervisor fields** — `SupervisorName` and `SupervisorDesignation` are copied text/choice at write time, not just references. This preserves the audit record if a Supervisors row is later edited or removed. `Supervisor` (the lookup) retains the link where one exists, for coordinator workflows.
- **`EntryDate` backdating cap** — students are informed at induction and in-app: "You can only log procedures performed in the last 7 days." Enforces near-real-time logging and prevents assessment-eve backfilling.

### 3.1 Outstanding schema changes (as at 12 Aug 2026)

SharePoint column changes — must be made by hand in the list settings. The Canvas Authoring MCP cannot create or alter SharePoint columns.

**1. 🔴 Add `SupervisorName` (single line of text) to `LogEntries_DEV` — blocking.**
The list currently relies on the `Supervisor: Title` lookup projection, which is **live, not copied**. Two consequences, both breaking the audit model:
  - Editing or deleting a `Supervisor_DEV` row retroactively rewrites the supervisor name on every historical entry that references it. The countersignature record silently changes after the fact.
  - Ad-hoc supervisors leave `Supervisor` blank by design, so the projection is blank too — the entry records **no supervisor name at all**. This breaks the ad-hoc path in PLAN.md Phase 3.

  The write flow must populate `SupervisorName` on every entry, from the registry record or the ad-hoc form, at write time. Do not delete the projection — it stays useful for coordinator views.

**2. 🟠 Convert `LogEntries_DEV.SupervisorDesignation` from Text to Choice.**
Options: Medical Doctor / Medical Intern / Professional Nurse / Enrolled Nurse / Clinical Associate / Other Health Professional — matching `Supervisor_DEV.Designation`, which is already a Choice. As free text the field cannot be reliably grouped or filtered for accreditation reporting, and app-side and registry-side values will drift apart. The list holds no production rows yet, so converting now costs nothing.

**3. 🟠 Confirm or remove `LogEntries_DEV.Performace` (Number).**
Not part of the design. If it is intended as a performance/competency rating it needs a defined scale and a decision on who sets it (student self-rating vs supervisor) before the write flow is built — an unlabelled numeric score is not defensible evidence. If it is a stray, delete it before any rows exist.

**4. 🟠 Delete `Procedure_DEV.RequiredCount0`.**
A duplicate of `RequiredCount`, displayed as "RequiredCount (RequiredCount0)". Two columns with near-identical display names is a live mis-binding risk for progress-count formulas. The 12 Aug 2026 export (`Procedure_DEV.csv`) contains a single `RequiredCount` column, populated on all 148 rows with no blanks — so `RequiredCount` is the real one and `RequiredCount0` appears to be an empty stray. Confirm it holds no data, then delete it.

**5. 🟡 Confirm `TSites_DEV.Radious` is in metres.**
The haversine site-match compares a metre distance against this value. The column name carries no unit. If it holds kilometres, every geofence check is wrong by 1000×.

**6. ✅ RESOLVED — `Procedure_DEV.Module` is a real grouping axis.** Keep it. See §3.2; the dashboard groups by `Module` and may filter by `Category`.

**7. 🟡 Verify `LogEntries_DEV` permissions.**
PLAN.md Phase 1 requires that students have **no direct write access**; writes come from the flow's service account only. This cannot be checked through the Canvas MCP — confirm in SharePoint list settings → Permissions.

**8. 🔴 Confirm `StudentList_DEV.Year` option text matches `Procedure_DEV.YearLevel` exactly — blocking Phase 4.**
Both are **Choice** columns, so the dashboard filter compares their option strings. `YearLevel` uses `"BMCP 1"` / `"BMCP 2"` / `"BMCP 3"` (confirmed from the 12 Aug 2026 export). If `Year` stores anything else — `1`, `"Year 1"`, `"BMCP1"` without the space — the filter silently returns **zero rows** and every student sees an empty dashboard with no error. Choice option text cannot be read through the Canvas MCP; verify in list settings. Either align the two option sets (preferred) or map between them explicitly in `OnStart`; do not rely on coincidence.

**9. 🟠 Delete the duplicate `"Perform general examination"` row in `Procedure_DEV` (BMCP 1).**
It appears **twice**, identical in every field. `LogEntries.Procedure` is a lookup by row ID, so the two rows are distinct targets: the procedure shows twice on the dashboard, each demanding its own 5 sign-offs, and logging against one never credits the other. Inflates BMCP 1 from 45 procedures / 245 sign-offs to 46 / 250. Delete one — but only before entries reference it.

**10. 🟠 Clarify `StudentList_DEV.Registered` (Boolean, display name "Dereg").**
The internal name and the display name assert **opposite** meanings, so `Registered = true` is genuinely ambiguous — it could mean "is registered" or "is deregistered". Any active-student filter written against it has a 50% chance of being exactly backwards, which would either hide active students or expose deregistered ones. Confirm the polarity against real rows and record it here before writing the filter.

### 3.2 Procedures content (as loaded, 12 Aug 2026)

Source of truth for what was loaded: `docs/Procedure_DEV.csv` (export of `Procedure_DEV`). Phase 0's "gather the procedure lists" item is **met**. All 148 rows have `Active = True` and no blank fields.

| Year (`YearLevel`) | Procedures | Total required sign-offs |
|---|---|---|
| `BMCP 1` | 46 *(45 after the duplicate in §3.1 item 9 is removed)* | 250 *(245 after)* |
| `BMCP 2` | 51 | 270 |
| `BMCP 3` | 51 | 237 |

`RequiredCount` is only ever **3**, **5** or **10** — 5 is the default, 10 marks high-volume core skills (history taking, vital signs, IV lines, wound care), 3 marks low-opportunity procedures (neonatal resuscitation, suprapubic catheterisation, posterior nasal packing).

**Two independent grouping axes.** Both are Choice columns and both are useful — they are not redundant:

- **`Category`** — *type of skill*, 4 options across all years: Physical Examination (58), Procedures (58), Communication (20), Investigations (12).
- **`Module`** — *clinical rotation / body system*. Years 1–2 share one taxonomy; **Year 3 uses a completely separate one, with no overlap**:
  - **BMCP 1 & 2:** Skills, Resp, CVS, GIT, GUT, CEE, ESR, MSS
  - **BMCP 3:** CH (Child Health), WH (Women's Health), A&P (Anaesthetics & Peri-operative), HCS (Health Care Systems), MH (Mental Health), INFD (Infectious Diseases), A&E (Accident & Emergency)

**Dashboard implication (Phase 4):** 46–51 procedures per year is far too long for a flat mobile gallery. Group by `Module` — that yields 7–8 sections of 4–12 items each, which is browsable one-handed. Treat `Category` as a filter, not the primary grouping.

**Deliberate cross-year repetition.** 12 procedure titles appear in both BMCP 1 and BMCP 2 (general examination, history taking, sensory/motor examination, urinary catheterisation, and others). This is intended progression, not duplication — each is a separate row with its own ID, so counts stay separate per year. One case escalates its target: *"Perform wound/burn debridement and dressing"* goes from 5 in Year 1 to 10 in Year 2.

> **Consequence for coordinator views:** the `Procedure` lookup in `LogEntries_DEV` displays only `Title`, so a repeated title gives no year context in SharePoint views. Add `YearLevel` as an additional lookup column if coordinators will read entries directly in SharePoint.

---

## 4. Open items (to resolve before/during build)

1. **Per-year procedure content** — the actual Year 1/2/3 procedure lists with required counts. Content only; cannot be inferred. Blocks the Procedures list.
2. **Ad-hoc supervisor policy** — allow students to add unregistered supervisors on the fly (flagged "Pending confirmation" for coordinator promotion), vs. coordinator pre-registration only. Leaning towards ad-hoc-with-flag for field practicality.
3. **Hospital geofence radii** — `RadiusMetres` per site; campuses vary in size.
4. **Coordinator visibility** — which emails see the coordinator view (site coordinators + programme lead).

---

## 5. Explicitly out of scope (for v1)

- Student card barcode scanning (belongs to another app).
- Email-based verification.
- Power BI dashboards (nice-to-have after core capture works).

- **Offline-first writes** / capturing entries with no signal and syncing later. Read-side caching of reference lists (§2.8) *is* in scope; only true offline *writes* are deferred, as they conflict with the flow-write integrity model (§2.6). Revisit if peripheral-site connectivity proves blocking.
