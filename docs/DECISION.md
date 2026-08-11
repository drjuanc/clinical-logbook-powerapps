# DECISION.md — Electronic Clinical Logbook

**Project:** Student clinical procedures logbook
**Platform:** Microsoft 365 tenant (Power Apps canvas + SharePoint + Power Automate)
**Status:** Design settled — ready for build
**Last updated:** 11 August 2026

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

---

## 3. Data model (settled)

| List | Status | Key columns |
|---|---|---|
| **Students** (existing `StudentList_DEV`) | Exists — no change | `Student` (Person, identity for `User().Email` match), `Number` (student number), `Name`, `Year`, `Hospital` (lookup), `Pic`, `Dereg*` |
| **Hospitals** | Exists — **add** `Latitude`, `Longitude`, `RadiusMetres` | Site registry + geofence |
| **Procedures** | New | `Title`, `YearLevel`, `RequiredCount`, `Category`, `Active` |
| **Supervisors** | New | `Title` (full name), `Designation` (Choice — see LogEntries §7), `CouncilNumber`, `Site` (lookup), `Status` (Registered/Pending) |
| **LogEntries** | New — flow-write only | See detailed schema below |

**DEV/PROD environments** are managed via environment variables. All list names in this project have a `_DEV` suffix in development and drop the suffix in production; solution references use variables, not hardcoded names, so cutover requires no rebuilding of Lookups or formulas.

### LogEntries — detailed schema (locked)

| # | Column | Type | Notes |
|---|---|---|---|
| 1 | Title | Text | Computed by the write flow, e.g. `"Ndlovu S · Suturing · 11 Aug 2026"` — makes SharePoint default views readable. Not the identifier. |
| 2 | Student | **Lookup → Students** | Primary display: `Number` (student number). Additional column: `Name`. Referential integrity enforced — cannot reference a non-existent student. Written by the flow using `varStudent.ID`. |
| 3 | Procedure | Lookup → Procedures | Referential integrity to Procedures. |
| 4 | SupervisionLevel | Choice | Observed / Assisted / Performed under supervision / Performed independently. |
| 5 | SupervisorRef | Lookup → Supervisors | Blank if ad-hoc (supervisor not yet in registry). |
| 6 | SupervisorName | Text | **Copied** from registry or ad-hoc entry at write time. Preserved even if the Supervisors row is later edited or deleted — this is the audit record. |
| 7 | SupervisorDesignation | Choice | **Medical Doctor / Medical Intern / Professional Nurse / Enrolled Nurse / Clinical Associate / Other Health Professional.** Copied same as SupervisorName. |
| 8 | SignatureImage | **Image column** | Written by the flow, which strips the `data:image/png;base64,` prefix from Pen Input, decodes to binary, and uploads. Renders as thumbnail in SharePoint views. |
| 9 | Latitude | Number | Raw GPS at moment of logging. Stored for audit; never displayed raw to students. |
| 10 | Longitude | Number | As above. |
| 11 | SiteMatch | Text | Human-readable: `"St Barnabas (140 m)"` or `"No site within radius"`. Computed by the app via haversine against the student's Hospital, passed to the flow. |
| 12 | LocationStatus | Choice | Captured / Unavailable. Blocks silent `(0,0)` storage; drives the "Flagged" status downstream. |
| 13 | EntryStatus | Choice | Valid / Flagged / Audit-verified. Defaults to Valid on write; auto-set to Flagged if `LocationStatus = Unavailable`; set to Audit-verified by the monthly sampling flow. |
| 14 | EntryDate | Date only | When the procedure was actually performed. Student-selectable. **Defaults to `Today()`, hard-blocked to the range `Today()-7 ≤ date ≤ Today()`** — no future, no backdating beyond 7 days. Distinct from SharePoint auto-`Created` timestamp; a gap between them is an audit signal. |
| 15 | Notes | Multiline text | Optional case context. **No patient identifiers** — enforced by app-side helper text + POPIA consent, not by SharePoint. |

Plus SharePoint's automatic system columns (do not add): `ID` (auto integer, the true identifier), `Created` (server timestamp — harder to spoof than a client-set date), `Created By` (will always be the write flow's service account, not the student), `Modified`, `Modified By`.

**Design rules embedded in this schema:**
- **Copied vs looked-up supervisor fields** — `SupervisorName` and `SupervisorDesignation` are copied text/choice at write time, not just references. This preserves the audit record if a Supervisors row is later edited or removed. `SupervisorRef` retains the link where one exists, for coordinator workflows.
- **`EntryDate` backdating cap** — students are informed at induction and in-app: "You can only log procedures performed in the last 7 days." Enforces near-real-time logging and prevents assessment-eve backfilling.

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
