# CLAUDE.md — project-template

AI context for the MATESTAIN `project-template` repository.
Read `.github/CLAUDE.md` first for org-level context.

---

## Purpose

This repo is the base template for all MATESTAIN client automation projects. It contains:
- The complete folder structure for any project
- Pre-built PLC, robot, and HMI base projects (per machine type)
- Pre-populated documentation templates
- Common reusable modules (`common/`)

Any client repo named `cc####-client-name` was created from this template.

---

## Repo Naming Convention

```
cc####-client-name
```

- `cc` — two-letter client code (consistent per client, registered in `company/pricing/client-registry.md`)
- `####` — 4-digit zero-padded sequential project number
- `client-name` — slugified short name (lowercase, hyphens, no spaces)

Examples: `cc0001-acme-dairy`, `cc0023-arcelor-eol`, `cc0101-nestle-packaging`

**All client repos are private. No exceptions.**

---

## Repo Structure

```
project-template/
├── common/                    — Shared modules (T-MOD-XXX), built once, never duplicated
│   ├── T-MOD-001_motor/
│   ├── T-MOD-008_estop/
│   ├── T-MOD-019_opcua-map/
│   └── ...
│
├── [machine-type]/            — One folder per machine template (T-STD-XXX)
│   ├── README.md              — Scope, hardware BOM, module list, template version
│   ├── CHECKLIST.md           — Pre-filled delivery checklist
│   ├── DESIGN.md              — Design document (paper design before TIA/RS5000)
│   ├── plc/
│   │   ├── tia-portal/        — Base .ap17 project + XML exports
│   │   └── studio-5000/       — Base .ACD project + L5X exports
│   ├── robot/                 — KUKA (.wvs + KRL) and Yaskawa (backup + .JBI)
│   ├── hmi/                   — WinCC Comfort/Unified, FT View ME/SE projects
│   └── docs/
│       ├── alarms/            — alarm-register.xlsx (pre-populated)
│       ├── electrical/        — io-list.xlsx
│       ├── safety/            — risk-assessment.xlsx, validation-plan.xlsx
│       ├── network/           — network-topology.xlsx, cybersecurity-handover.md
│       ├── mapping/           — tag-xref.xlsx
│       ├── reports/           — commissioning-log.xlsx
│       └── operations/        — operator-manual-EN.docx, training-presentation-EN.pptx,
│                                competency-checklist-EN.xlsx, operator-exam-EN.docx
│
└── TEMPLATE-MASTER-LIST.md    — Registry of all templates, modules, status, cross-reference
```

When creating a client project from a template, copy the relevant `[machine-type]/` folder and rename it following the project structure in `company/standards/project-structure.md`.

---

## Standards Reference

All engineering decisions follow the standards in `company/standards/`. When Claude makes a decision about naming, structure, alarms, safety, or network configuration, it must check the relevant standard first.

| Standard | Key decisions it governs |
|---|---|
| `naming-conventions.md` | Tag names, block names, file names, all identifiers |
| `project-structure.md` | Folder layout inside client repos |
| `version-control.md` | Git workflow, commit messages, branching, export procedure |
| `alarm-management.md` | Alarm IDs, domain ranges, severity levels, buffer profiles |
| `support-documents.md` | Which support files are mandatory, column specs |
| `safety.md` | Minimum PL by machine type, non-participation conditions, code structure |
| `network-standards.md` | IP allocation, PROFINET naming, OPC-UA config, remote access policy |
| `hmi-scada.md` | Color palette, typography, navigation model, faceplate design |
| `management-of-change.md` | MOC classification, pre/post-FAT process, commercial triggers |
| `delivery-checklist.md` | Final gate before project delivery |
| `template-standards.md` | Hardware families, software versions, programming language policy |

**Short version of the most-used rules:**

- **Naming:** `AREA_DEVICE_SIGNAL` for tags (e.g., `VALV_V01_FBK_OP`). PascalCase for block names (e.g., `FB_VALV_OnOff`). English only.
- **Language:** LAD default. SCL only for math, string handling, array processing, or drive library blocks. FBD never.
- **Commits:** `type(scope): description` — e.g., `feat(tia/valv): add valve timeout fault logic`
- **Versions:** Semantic versioning. `0.x.x` pre-FAT, `1.0.0` at FAT sign-off.
- **Alarms:** Format `[Severity][Domain]-[Index]` e.g., `F1-0023`. Domain 1=PLC, 2=Robot, 3=Drive, etc.

---

## Template Module Registry

All common modules are documented in `TEMPLATE-MASTER-LIST.md`. Before building any block:
1. Check if it already exists in `common/`
2. If not, check if it's planned (status `⬜`) — add context before building
3. Build in `common/`, never inside a machine-type folder

Module IDs: `T-MOD-XXX`. Machine template IDs: `T-STD-XXX`.

---

## Required Files in Every Client Repo

Before any work is committed:

- `docs/scope.md` — agreed scope, deliverables, out-of-scope, timeline, change order history
- `docs/contacts.md` — client PM, site supervisor, emergency contact, MATESTAIN team
  - **`contacts.md` must be in `.gitignore`** — never committed to remote
- `docs/commercial/` — NDA, Technical Offer, Commercial Quote, Service Agreement, POs, Invoices
  - Templates in `brand/templates/proposal/src/` — copy, fill placeholders, export to PDF
  - Filled PDFs are transmitted via secure channel, **not committed to Git** (add `docs/commercial/export/` to `.gitignore`)
  - Naming: `[DOCTYPE]-[CLIENTCODE]-[YEAR]-[NNN]-v[x.y].md`
  - Client code registered in `company/pricing/client-registry.md`

---

## What Not to Do

- Do not make a client repo public
- Do not commit `contacts.md` to any remote
- Do not start work before `scope.md` is written
- Do not rename repos mid-project
- Do not use FBD in any PLC program
- Do not create tags or blocks with Spanish names
- Do not implement an alarm in code without first adding it to the alarm register
- Do not assign an IP address outside the scheme in `network-standards.md`
- Do not begin post-FAT changes without written client authorization

---

## Commit Conventions

Follow Conventional Commits with project scope:

```
feat(cc0001/tia/valv): add valve cluster sequence logic
fix(cc0023/rs5000/palz): correct layer count reset on recipe change
tune(cc0101/kuka): reduce approach speed to 60% on layer 5
export(cc0001/tia): automated XML export v0.3.0
docs(cc0001): update scope with change order 2
chore(template): update valve cluster small file structure
```

---

_Part of the MATESTAIN organization — see `.github/CLAUDE.md` for org-wide context._
