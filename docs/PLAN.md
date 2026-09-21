# PLAN.md — Electronic Clinical Logbook Build

**Companion to:** DECISION.md
**Build via:** Claude Code + PowerApp MCP
**Sequencing principle:** Build in dependency order — data before logic, logic before UI. Nothing that reports location works until the geofence exists; nothing writes safely until the flow exists.

**Cross-cutting constraints (apply to every UI phase — see DECISION.md §2.7–2.8):**
- **Mobile-first, container layout:** every screen built with a root vertical container + horizontal containers for rows. No absolute X/Y positioning. Portrait phone is the design target; must reflow, not break, on tablet.
- **One-handed use:** primary action (Submit) reachable near the bottom; thumb-sized tap targets; single-column, no horizontal scroll.
- **Light + fast on poor networks:** cache slow-changing reference lists (Procedures, Supervisors, Hospitals) into collections at startup; parallelise loads with `Concurrent()`; keep `OnStart` minimal; never pull the full LogEntries list to the device; don't re-fetch signature images for browsing; prefer collections/variables over repeated live queries.

---

## Locked constraints (apply to EVERY phase)

These are hard requirements from DECISION.md §2.7–2.8. Check each screen against them.

- **Mobile-first** — phone is the primary target; design for it first.
- **Responsive via containers** — all layout uses horizontal/vertical containers, not absolute positioning. Screens reflow to width.
- **Lightweight** — few controls per screen, minimal images, reference data cached once at start, non-essential data deferred.
- **Fast on poor networks** — cache slow-changing reference lists (Procedures, Supervisors, Hospitals) into collections at startup; parallelise with `Concurrent()`; minimal `OnStart`; never pull the full LogEntries list; don't re-fetch signature images for browsing.

---

## Phase 0 — Prerequisites (content + admin)

- [x] Gather Year 1/2/3 procedure lists with `RequiredCount` per procedure (blocks Phase 2). *(Done — 148 procedures loaded and exported to `docs/Procedure_DEV.csv`; taxonomy and volumes in DECISION.md §3.2.)*
- [ ] Confirm `Latitude`/`Longitude`/`RadiusMetres` for each hospital (blocks Phase 1 site-match).
- [ ] Provision a **service account** for the write flow.
- [ ] Confirm coordinator email addresses for conditional visibility.
- [ ] Draft the POPIA logbook clause + student consent wording (location + signature processing).

---

## Phase 1 — Data foundation (SharePoint)

*Status 12 Aug 2026: lists built and connected to the canvas app. As-built schema verified via the Canvas Authoring MCP and recorded in DECISION.md §3 — actual list names are `TSites_DEV`, `Procedure_DEV`, `Supervisor_DEV`, `LogEntries_DEV`, `StudentList_DEV`.*

- [x] Add `Latitude` (Number), `Longitude` (Number), `RadiusMetres` (Number) to the existing **Hospitals** list (`TSites_DEV`); populate. *(Built as `Latitude`, `Longitude`, `Radious` — confirm the unit is metres, DECISION.md §3.1 item 5.)*
- [x] Create **Procedures** list (`Procedure_DEV`): `Title`, `YearLevel` (Choice 1/2/3), `RequiredCount` (Number), `Category` (Choice), `Active` (Yes/No). *(Plus an extra `Module` Choice; a stray `RequiredCount0` duplicate needs deleting — §3.1 items 4 and 6.)*
- [x] Create **Supervisors** list (`Supervisor_DEV`): `Title` (full name), `Designation` (Choice), `CouncilNumber` (Text), `Site` (Lookup→TSites), `Status` (Choice: Registered/Pending). *(Matches the design.)*
- [~] Create **LogEntries** list (`LogEntries_DEV`) with the full schema from DECISION.md §3. **Incomplete —** `SupervisorName` (Text) is missing and `SupervisorDesignation` is Text rather than Choice. See DECISION.md §3.1 items 1–3. **Blocks Phase 3.**
- [ ] Set **LogEntries permissions**: no direct student write access (writes come from the flow's service account only). *(Not verifiable through the Canvas MCP — check in SharePoint list settings.)*

**Exit test:** every list exists; a manually added test LogEntries row displays all columns correctly. — *Not yet met: the LogEntries schema gaps above must close first.*

---

## Phase 2 — Content load

- [x] Populate **Procedures** from the Year 1/2/3 lists. *(148 rows: BMCP 1 = 46, BMCP 2 = 51, BMCP 3 = 51. One duplicate to remove — DECISION.md §3.1 item 9.)*
- [ ] Seed **Supervisors** with known internal staff and any pre-registered externals. *(Row count not verified — the Canvas MCP cannot read list data.)*

**Exit test:** filtering Procedures by `YearLevel` returns the correct set per year. — *Not yet met.* `YearLevel` uses `"BMCP 1/2/3"`; the filter compares it against the `StudentList_DEV.Year` Choice, whose option text is unconfirmed. A mismatch returns zero rows **silently**. See DECISION.md §3.1 item 8.

> **Caching note:** this reference data (student record, year's Procedures, hospital coordinates/radius, site Supervisors) is what gets preloaded into collections at startup in Phase 4. Keep these lists lean — everything cached is weight the app carries and bandwidth it costs on first load.

---

## Phase 3 — Submission flow (Power Automate)

- [ ] Build the **write flow** (service-account context) triggered from the app (PowerApps V2 trigger).
- [ ] Inputs: student identity, procedure, supervision level, supervisor fields, signature (data URI), lat/long, site match, location status.
- [ ] Convert Pen Input data URI → image file; store against the entry (`SignatureImage` column or document library keyed by entry ID).
- [ ] Write the LogEntries row; return the new record ID to the app.
- [ ] Handle ad-hoc supervisor: if not in registry, create a Supervisors row with `Status = Pending`.

**Exit test:** calling the flow with test data creates a LogEntries row with a viewable signature image.

---

## Phase 4 — Canvas app: identity, caching + dashboard

*Status 12 Aug 2026 (commit `6091c1e`, branch `feat/identity-splash-theme`): identity flow and theme built and pushed to the coauthoring session. Dashboard and caching not started. Source in `src/`.*

- [x] **Design theme** — palette sampled from `docs/design/WSU_new_logo.png`, WCAG AA verified, exposed as named formulas under `App.Properties.Formulas`. Screens reference tokens only, never raw hex. Documented in `docs/THEME.md`. *(Not in the original plan; added because every subsequent screen depends on it.)*
- [~] Set the app to a **phone form factor**; scaling to fit off; build all screens with a root **vertical container** holding **horizontal containers** per row. No absolute positioning. *(Containers done — zero absolute positioning in `src/`. **Form factor and scale-to-fit not yet verified** in Studio app settings.)*
- [x] On start: `Set(varStudent; LookUp('Students'; Student.Email = User().Email))`. *(Built as `LookUp(StudentList_DEV, Student.Email = User().Email)` in `scrSplash.OnVisible`, not `App.OnStart` — `App.StartScreen` evaluates before `OnStart` so it cannot branch on the result, and running it on the splash makes the loading state visible. **Not delegable** — see the warning below.)*
- [ ] **Preload reference data into collections** at start, in parallel via `Concurrent()`: the student's year Procedures (`colProcedures`), their site's Supervisors (`colSupervisors`), Hospitals coordinates (`colHospitals`). Read from these collections thereafter, not the live lists. *(Optional: `SaveData`/`LoadData` for faster warm starts — cache only, not an offline write path.)*
- [x] Keep `OnStart` lean — load only what the first screen needs; defer the rest to their screen. *(`App.OnStart` is now empty; App Checker flagged the one variable it held as unused.)*
- [ ] Add a manual **"Refresh reference data"** control that re-runs the collection load.
- [x] Handle "student not found" gracefully (not registered / deregistered). *(`scrNotRegistered` — plain language, no error codes, shows the signed-in email. Deregistration is detected via `IsBlank(Dereg_Date)`, **not** the ambiguous Boolean — DECISION.md §3.1 item 10.)*
- [ ] **Home/Dashboard screen:** vertical container → gallery of the student's year procedures with logged-vs-required counts and a progress bar. Count via `CountRows(Filter(LogEntries; Student.Email = User().Email; Procedure.Id = ThisItem.ID))` — filtered server-side to the current student only.
- [~] Show `varStudent.Name` and `varStudent.Pic` (keep image light — small/thumbnail). *(`Name`, `Number`, `Year.Value` and `Hospital.Value` render on `scrHome`. `Pic` deferred — the schema shows it as a Record and the access pattern is unverified.)*

**Exit test:** a test student sees only their year's procedures with correct counts, and the layout reflows cleanly on rotate/resize without absolute positioning. — *Not met: the procedure gallery is not built. `scrHome` is a placeholder card that only proves the identity lookup resolved.*

> ⚠️ **Blocking risk carried into this phase — the identity lookup is not delegable.** `Student` is a Person column, so `Student.Email = User().Email` cannot be delegated to SharePoint: Power Apps pulls up to the data row limit (500 by default) and matches on the device. **Students beyond that row silently land on "not registered"** — it will pass testing and fail in production. The durable fix is a plain Text email column on `StudentList_DEV`, which *is* delegable. Raising the limit to 2000 buys time, not a fix.

> ⚠️ **The dashboard filter is the next thing that breaks.** `Procedure_DEV.YearLevel` holds `"BMCP 1"` / `"BMCP 2"` / `"BMCP 3"`. If `StudentList_DEV.Year`'s choice text differs by even a space, the filter returns zero rows **with no error** and every student sees an empty dashboard. Confirm before building the gallery — DECISION.md §3.1 item 8.

---

## Phase 5 — Canvas app: entry + verification

- [ ] **New Entry screen:** procedure (pre-filtered to year), supervision level, date (default today), notes.
- [ ] **Supervisor + Signature screen:**
  - [ ] Registry search filtered to `varStudent.Hospital`; "Not listed? Add supervisor" reveals name/surname/designation.
  - [ ] **Pen Input** signature control.
  - [ ] Capture GPS silently; show indicator ("📍 St Barnabas — 140 m").
  - [ ] Compute `SiteMatch` via **haversine** against the student's hospital lat/long + radius.
  - [ ] Disable Submit until signature has ink AND location resolves — or student explicitly acknowledges "location unavailable" (flags entry).
  - [ ] **On Submit** → call the Phase 3 write flow, show a lightweight success state, then update the dashboard count. Disable the button while the call is in flight to prevent double submission on a slow connection.
- [ ] Lay out both screens with vertical/horizontal containers so the form reflows on any phone width.

**Exit test:** a full bedside entry — procedure → supervisor → signature → GPS → submit — lands correctly in LogEntries and updates the dashboard count.

---

## Phase 6 — Read + coordinator views

- [ ] **My Logbook screen:** read-only, filterable history for student self-check.
- [ ] **Coordinator view** (conditional on email): pending supervisor confirmations, flagged entries, per-student progress at the coordinator's site.

**Exit test:** coordinator sees pending supervisors and can promote them to Registered.

---

## Phase 7 — Assurance + polish

- [ ] **Monthly audit-sampling flow:** randomly sample entries per site, email the site coordinator a shortlist to spot-verify.
- [ ] POPIA consent screen on first app launch.
- [ ] Error/empty states; deregistered-student handling; GPS-denied guidance.

**Exit test:** audit flow runs on schedule and emails a valid sample.

---

## Deferred (post-v1)

- Power BI accreditation dashboards (completion %, students falling behind, per-site activity).
- Shared "station device" mode using the student card barcode (other app's territory — revisit only if needed).
- Offline-first capture for poor-connectivity sites.

---

## Key formulae to implement (reference)

- **Student identity:** `Set(varStudent; LookUp('Students'; Student.Email = User().Email))`
- **Progress count:** `CountRows(Filter(LogEntries; Student.Email = User().Email; Procedure.Id = ThisItem.ID))`
- **Haversine site match:** distance from `Location.Latitude/Longitude` to `varStudent.Hospital` coordinates; compare against `RadiusMetres` → format as `"<Site> (<distance> m)"` or `"No site within radius"`.
- **Location guard:** block/flag when `LocationStatus = Unavailable` rather than storing `(0,0)`.
