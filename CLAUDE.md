# CLAUDE.md — project-template

AI context for the MATESTAIN `project-template` repository. Read the org-level `.github/CLAUDE.md` first.

---

## Purpose

This is the base template for all MATESTAIN client automation projects. Any repo named `cc####-client-name` was created from this template.

## Naming Convention

Client repos follow a strict naming pattern:

```
cc####-client-name
```

- `cc` — two-letter client code prefix (consistent per client)
- `####` — 4-digit zero-padded sequential project number
- `client-name` — slugified short name (lowercase, hyphens, no spaces)

This naming is used for directory organization, billing reference, and file archiving. Do not deviate from it.

## Visibility

**All client project repos are private.** No exceptions. Client data, site contacts, PLC source code, and project scope are confidential by default.

If a client explicitly authorizes open-sourcing a deliverable, that deliverable gets its own separate public repo — it does not make the client project repo public.

## Required Documents

Every client repo must have these two files before any work is committed:

### `docs/scope.md`

Defines the project in writing:
- What was agreed with the client
- List of deliverables
- What is explicitly out of scope
- Timeline and milestones (if applicable)
- Change order history (append, never edit previous entries)

### `docs/contacts.md`

Operational contacts for the project:
- Client-side project manager
- Site supervisor / operator contact
- Emergency or after-hours contact
- MATESTAIN team members assigned

`contacts.md` must be listed in `.gitignore` to prevent it from being shared if the repo is ever forked or cloned outside the org. It should exist in the working tree but never be pushed to a public remote.

## What Not to Do

- Do not make a client repo public
- Do not commit `contacts.md` to a shared or public remote
- Do not start work before `scope.md` is written and reviewed
- Do not rename repos mid-project — the name is used for billing and archiving

## Commit Conventions

Follow org-level Conventional Commits. Use the project code as scope:
```
feat(cc0001): add filling line PLC sequence
fix(cc0023): correct end-of-line reject logic
docs(cc0101): update scope with change order 2
```

---

_Part of the MATESTAIN organization — see `.github/CLAUDE.md` for org-wide context._
