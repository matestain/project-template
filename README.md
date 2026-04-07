# project-template

Base template for MATESTAIN client automation projects.

## How to use

When starting a new client project:

1. Copy this repo (or use it as a GitHub template)
2. Rename it following the convention: `cc####-client-name`
3. Make it **private** immediately
4. Fill in `docs/scope.md` and `docs/contacts.md` before anything else

## Naming Convention

```
cc####-client-name
```

- `cc` — client code prefix (always two letters)
- `####` — sequential 4-digit project number (e.g., `0001`, `0042`)
- `client-name` — short, slugified client or project name

Examples: `cc0001-acme-dairy`, `cc0023-arcelor-eol`, `cc0101-nestle-pakaging`

## Repo Structure

```
cc####-client-name/
├── docs/
│   ├── scope.md        # Project scope and deliverables (required)
│   └── contacts.md     # Client contacts — keep this gitignored in forks
├── src/                # Source code, PLC programs, scripts
├── assets/             # Drawings, diagrams, photos
└── README.md
```

## Required files

Every client project repo must have:

- `docs/scope.md` — what was agreed, what will be delivered, what is out of scope
- `docs/contacts.md` — client contacts, site contacts, emergency contacts

See templates in `docs/` for the expected format.

---

_Maintained by Lucas Viera — lucas@matestain.com_
