# CLAUDE.md — Electronic Clinical Logbook

Operational guide for Claude Code. Read `DECISION.md` (settled architecture + rationale) and `PLAN.md` (phased build) before acting — they are the source of truth. This file is the short version plus the guardrails.

---

## What this is

An electronic clinical logbook that tracks the procedures students perform across **Years 1, 2 and 3**. Each year has its own list of required procedures with a target count per student. Supervisors countersign each entry. Output must be defensible evidence for **HPCSA accreditation**.

## Stack

- **Front-end:** Power Apps **canvas** app (mobile-first).
- **Data:** SharePoint Lists (M365 tenant).
- **Logic:** Power Automate flows.
- **Built via:** Claude Code + PowerApp MCP. Standard connectors only — **no premium licensing**.

---

## Non-negotiable guardrails

Do not violate these. If a request conflicts with one, stop and flag it rather than complying silently.

1. **Students never write to `LogEntries` directly.** All writes go through a Power Automate flow running under a **service account**. Never wire the app to patch `LogEntries` from the client.
2. **Mobile-first, container layout only.** Root **vertical container** + **horizontal containers** for rows. **No absolute X/Y positioning.** Portrait phone is the design target; must reflow, not break, on tablet.
3. **Light + fast on poor networks.** Preload slow-changing lists (Procedures, Supervisors, Hospitals) into collections at startup; parallelise with `Concurrent()`; keep `OnStart` minimal; never pull the full `LogEntries` list to the device (filter server-side to the current student); don't re-fetch signature images for browsing.
4. **No offline write-queue.** Caching reference data for speed is in scope. Capturing entries offline and syncing later is **explicitly out of scope for v1** — it conflicts with the flow-write model (a flow can't run offline). Do not add it unless the human explicitly asks and the conflict is resolved first.
5. **POPIA.** No patient identifiers in any field, including `Notes`. Location is captured **only at the moment of logging** — never continuous tracking. Signature + location processing requires student consent wording.
6. **Verification = name + surname + designation + signature (Pen Input) + GPS + timestamp.** This is final. Do **not** re-introduce ruled-out approaches: AAD guest accounts for externals, supervisor PINs, or email-verification links (see DECISION.md §2.3).
7. **GPS failure flags the entry** (`LocationStatus = Unavailable`) — never silently store `(0,0)`.

---

## Identity model

- **Students** list already exists — reuse it, do not recreate it.
- Identify the logged-in student via the **`Student`** column (a **Person** column) against `User().Email`:
  `Set(varStudent; LookUp('Students'; Student.Email = User().Email))`
- Everything flows from `varStudent`: `.Year` filters procedures, `.Hospital` drives the site-match + supervisor registry, `.Name`/`.Pic` dress the UI.
- The **`Title`** column is the **student card barcode** — used by a *different* app. **Not used here.** Do not build barcode logic into this app.
- Active/inactive comes from `Dereg`/`Dereg_Date`/`Dereg_reason` — no separate `Active` flag on students.

---

## Data model (see DECISION.md §3 for full columns)

| List | Status |
|---|---|
| **Students** | Exists — no change |
| **Hospitals** | Exists — add `Latitude`, `Longitude`, `RadiusMetres` |
| **Procedures** | New |
| **Supervisors** | New (ad-hoc additions land as `Status = Pending`) |
| **LogEntries** | New — **flow-write only** |

`LogEntries.SupervisorName` is **copied text**, not just a lookup, so history survives registry edits.

---

## Conventions

- **Collections:** `col` prefix — `colProcedures`, `colSupervisors`, `colHospitals`.
- **Variables:** `var` prefix — `varStudent`.
- **Controls:** meaningful prefixes (`ddProcedure`, `btnSubmit`, `penSignature`, `galProcedures`).
- **Layout:** every screen = one root vertical container; rows = horizontal containers; use `FillPortions`, not pixel offsets.
- **Reads:** from collections/variables, not repeated live `LookUp`/`Filter` that hit the network each time.
- **Writes to LogEntries:** via the write flow only.

---

## Reference formulae

- **Identity:** `Set(varStudent; LookUp('Students'; Student.Email = User().Email))`
- **Progress count:** `CountRows(Filter(LogEntries; Student.Email = User().Email; Procedure.Id = ThisItem.ID))`
- **Site match:** haversine from `Location.Latitude/Longitude` to `varStudent.Hospital` coords; compare to `RadiusMetres`; store readable `"<Site> (<distance> m)"` or `"No site within radius"`.

---

## How to work

- **Follow PLAN.md phase order** — data → content → flow → app. Don't build UI against lists that don't exist yet.
- **Each phase has an exit test in PLAN.md** — meet it before moving on.
- **Ask before:** creating/altering SharePoint columns, changing the verification model, touching permissions, or anything that would give students write access.
- **Don't invent the procedure content** — the Year 1/2/3 procedure lists + required counts come from the human (PLAN.md Phase 0). Flag if missing rather than guessing.
- **Check MCP tool coverage first:** the PowerApp MCP may not create SharePoint lists — if list/schema creation is out of its scope, say so and hand those steps back to the human rather than faking them.
- **Prefer small, verifiable steps** over large speculative builds.

---

## Out of scope (v1)

Barcode scanning · email verification · offline write-queue · Power BI dashboards. Don't build these unless asked.
