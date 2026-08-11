# Clinical Logbook — Power Apps

An electronic clinical logbook for health sciences students, built on a standard **Microsoft 365 tenant** — no premium licensing required. Replaces paper logbooks with a mobile-first app that tracks year-specific procedures and captures supervisor sign-off with signature + geolocation.

**Status:** planning + wireframes complete; build in progress.

---

## What it does

- Students see only **their year's required procedures** and live progress toward each target.
- Each entry is verified at the bedside by a supervisor's **name, designation, signature** (drawn on the student's phone), plus **GPS** and timestamp.
- Supervisors can be external (rotating nurses, sessional doctors) with no institutional account — verification does not depend on their login.
- Coordinators get a per-site view of pending supervisors, flagged entries, and students falling behind.
- Designed for **rural / low-bandwidth** use: mobile-first, containerised responsive layout, aggressive caching of reference data.

## Stack

- **Front-end:** Power Apps canvas app (mobile-first)
- **Data:** SharePoint Lists
- **Logic:** Power Automate flows (writes routed via a service account for integrity)
- **Tenant:** any standard Microsoft 365 tenant — no premium connectors

## Documentation

Planning docs live in `docs/`:

- **[`DECISION.md`](docs/DECISION.md)** — settled architecture decisions and the rationale (including what was ruled out and why).
- **[`PLAN.md`](docs/PLAN.md)** — phased build plan with exit tests per phase.
- **[`WIREFRAME.md`](docs/WIREFRAME.md)** — per-screen layout specifications.
- **[`design/`](docs/design/)** — wireframe images.

The [`CLAUDE.md`](CLAUDE.md) at the root is the operational guide loaded by [Claude Code](https://www.anthropic.com/claude-code) — it summarises the guardrails and points at the docs above.

## Design intent

- **Mobile-first** — designed for a student's phone at the bedside; responsive up to tablet, never optimised for desktop.
- **Container-based layout** — every screen built with vertical + horizontal containers, no absolute positioning.
- **Light on the network** — reference data cached at startup; per-action writes minimised.
- **POPIA-conscious** — no patient identifiers stored; location captured only at the moment of logging.
- **HPCSA-defensible** — signature + GPS + timestamp mirrors the paper wet signature with better metadata.

## Contributing

Issues and PRs welcome, particularly from other health sciences programmes adapting this pattern. If you're building on top of this in your own institution, I'd love to hear about it.

## Licence

MIT — see [`LICENSE`](LICENSE). Use it, fork it, adapt it. Attribution appreciated but not required for internal institutional use.
