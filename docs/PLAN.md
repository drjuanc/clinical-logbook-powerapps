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

- [ ] Gather Year 1/2/3 procedure lists with `RequiredCount` per procedure (blocks Phase 2).
- [ ] Confirm `Latitude`/`Longitude`/`RadiusMetres` for each hospital (blocks Phase 1 site-match).
- [ ] Provision a **service account** for the write flow.
- [ ] Confirm coordinator email addresses for conditional visibility.
- [ ] Draft the POPIA logbook clause + student consent wording (location + signature processing).

---

## Phase 1 — Data foundation (SharePoint)

- [ ] Add `Latitude` (Number), `Longitude` (Number), `RadiusMetres` (Number) to the existing **Hospitals** list; populate.
- [ ] Create **Procedures** list: `Title`, `YearLevel` (Choice 1/2/3), `RequiredCount` (Number), `Category` (Choice), `Active` (Yes/No).
- [ ] Create **Supervisors** list: `Title` (full name), `Designation` (Choice), `CouncilNumber` (Text), `Site` (Lookup→Hospitals), `Status` (Choice: Registered/Pending).
- [ ] Create **LogEntries** list with the full schema from DECISION.md §3.
- [ ] Set **LogEntries permissions**: no direct student write access (writes come from the flow's service account only).

**Exit test:** every list exists; a manually added test LogEntries row displays all columns correctly.

---

## Phase 2 — Content load

- [ ] Populate **Procedures** from the Year 1/2/3 lists.
- [ ] Seed **Supervisors** with known internal staff and any pre-registered externals.

**Exit test:** filtering Procedures by `YearLevel` returns the correct set per year.

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

- [ ] Set the app to a **phone form factor**; scaling to fit off; build all screens with a root **vertical container** holding **horizontal containers** per row. No absolute positioning.
- [ ] On start: `Set(varStudent; LookUp('Students'; Student.Email = User().Email))`.
- [ ] **Preload reference data into collections** at start, in parallel via `Concurrent()`: the student's year Procedures (`colProcedures`), their site's Supervisors (`colSupervisors`), Hospitals coordinates (`colHospitals`). Read from these collections thereafter, not the live lists. *(Optional: `SaveData`/`LoadData` for faster warm starts — cache only, not an offline write path.)*
- [ ] Keep `OnStart` lean — load only what the first screen needs; defer the rest to their screen.
- [ ] Add a manual **"Refresh reference data"** control that re-runs the collection load.
- [ ] Handle "student not found" gracefully (not registered / deregistered).
- [ ] **Home/Dashboard screen:** vertical container → gallery of the student's year procedures with logged-vs-required counts and a progress bar. Count via `CountRows(Filter(LogEntries; Student.Email = User().Email; Procedure.Id = ThisItem.ID))` — filtered server-side to the current student only.
- [ ] Show `varStudent.Name` and `varStudent.Pic` (keep image light — small/thumbnail).

**Exit test:** a test student sees only their year's procedures with correct counts, and the layout reflows cleanly on rotate/resize without absolute positioning.

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
