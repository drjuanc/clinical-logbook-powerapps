# WIREFRAME.md — Screen specifications

Derived from agreed wireframes (11 Aug 2026). Build each screen to match this structure. All screens: phone form factor, root **vertical container**, rows as **horizontal containers**, no absolute positioning. Primary action always near the bottom.

**Navigation:** bottom bar with 3 tabs — Home / Logbook / Profile. A 4th tab, **Coordinator**, appears only when `User().Email` is in the coordinator set.

---

## Screen 1 — Home / Dashboard (`scrHome`)

Vertical container, top to bottom:

1. **Header row** (horizontal): avatar image (`varStudent.Pic`, small thumbnail) · name + "Year X · Hospital" subtitle (`varStudent`) · spacer · refresh icon (`icoRefresh`) that re-runs the reference-data collection load.
2. **Overall progress card**: label "Overall progress" · large "N / M procedures logged" · thin progress bar (accent).
3. **Section label**: "Your procedures — Year X".
4. **Procedure gallery** (`galProcedures`, source `colProcedures`): one row per procedure —
   - Horizontal: procedure name (left) · "logged / required" count (right; success colour + full bar when complete, accent otherwise).
   - Thin progress bar under the row.
5. **Primary button** (`btnNewEntry`): "Log a procedure" — full width, near bottom, navigates to `scrEntry`.
6. **Bottom nav**.

Counts: `CountRows(Filter(LogEntries; Student.Email = User().Email; Procedure.Id = ThisItem.ID))` — server-side filtered to the current student.

---

## Screen 2 — New entry, step 1 of 2 (`scrEntry`)

Header row: back arrow · "New entry" · "Step 1 of 2" (right, muted).

1. **Procedure** (`ddProcedure`): dropdown, source `colProcedures` (already year-filtered). Pre-selected if user tapped a specific procedure row on Home.
2. **Supervision level** (`chpSupervision`): 2×2 grid of tap chips — Observed / Assisted / Under supervision / Independent. Single-select; selected chip gets accent border + accent text.
3. **Date** (`dtpDate`): date picker, defaults to today.
4. **Notes** (`txtNotes`): multiline, optional. Inline label hint: "— no patient identifiers".
5. **Primary button** (`btnToVerify`): "Continue to verification" → `scrVerify`. Disabled until procedure + supervision level chosen.

---

## Screen 3 — Verification, step 2 of 2 (`scrVerify`)

The hand-the-phone-over screen. Header row: back arrow · "Verification" · "Step 2 of 2".

1. **Supervisor search** (`txtSupSearch` + `galSupervisors`): search box "Search <Hospital> staff…", results from `colSupervisors` (pre-filtered to `varStudent.Hospital`). Selected supervisor renders as an accent-tinted card: name + designation + check icon.
2. **Ad-hoc link** (`lblAddSup`): "+ Not listed? Add supervisor" — reveals `txtSupName`, `txtSupSurname`, `ddSupDesignation`. Ad-hoc entries create a Supervisors row with `Status = Pending` (via the write flow).
3. **Signature** (`penSignature`): Pen Input in a dashed-border box, "Clear" affordance, caption below: "Hand the phone to your supervisor to sign".
4. **Location banner** (`lblLocation`): captured silently on screen entry.
   - Success (green tint): pin icon + "Location captured — <Site> (<distance> m)" from the haversine match.
   - Failure (warning tint): "Location unavailable — entry will be flagged" + explicit acknowledge tap required.
5. **Primary button** (`btnSubmit`): "Submit entry". Disabled until: supervisor chosen (registry or ad-hoc complete) AND signature has ink AND location resolved-or-acknowledged. On tap: disable while in flight (prevent double submit on slow networks) → call the write flow → success state → return Home.

---

## Screen 4 — My logbook (`scrLogbook`)

Read-only. No edit affordances anywhere.

1. **Title**: "My logbook".
2. **Filter chips** (horizontal): All (default) · by procedure · by period.
3. **Entry gallery** (`galEntries`, newest first, server-side filtered to current student): each row —
   - Horizontal: procedure name (500 weight) · date (right, muted).
   - Subtitle: "SupervisorName · SiteMatch".
   - Status badge from `EntryStatus`: Valid (success tint) · Flagged (warning tint) · Audit-verified (success tint).
4. **Bottom nav**.

---

## Screen 5 — Coordinator (`scrCoordinator`)

Visible only to coordinator emails. Scope: entries/students at the coordinator's site.

1. **Header**: "Coordinator — <Site>" + "N students at this site" subtitle.
2. **Metric cards** (2-up grid): Flagged entries (warning number, taps through to a flagged-entry list) · Pending supervisors (accent number).
3. **Pending supervisor confirmations** (`galPendingSups`): each row — name + "Added by student · date" · Reject (outline) + Register (primary) buttons. Register flips `Status` to Registered.
4. **Student progress** (`galStudentProgress`): each row — student name · mini progress bar · percentage. Below-threshold students render in warning colour.

---

## Screen 6 — Profile (`scrProfile`) — thin, settled furniture

Student details (name, number, year, hospital, photo) · POPIA consent status · "Refresh reference data" button · app version. No editing of student data in-app.

---

## Design decisions locked with these wireframes

- **Two-step entry flow** (details → verification), not one long scroll — makes the phone-handover moment explicit and keeps screens light.
- **Supervision level = tap chips**, not a dropdown.
- **Bottom nav**, 3 tabs + conditional Coordinator.
- **Students see their own status badges** (incl. Flagged) — transparency drives self-correction.
