# docs/commercial/

Client-facing commercial documents for this project. Templates live in `brand/templates/proposal/`.

---

## Setup (once per project)

```bash
# Copy the project.yaml template here and fill it
cp brand/templates/proposal/src/project.yaml docs/commercial/project.yaml
# Edit docs/commercial/project.yaml — client info, project details, commercial terms
```

---

## Generate a document

```bash
python brand/templates/proposal/src/_assets/export.py \
  --template brand/templates/proposal/src/01-nda.md \
  --project  docs/commercial/project.yaml

# With runtime overrides
python brand/templates/proposal/src/_assets/export.py \
  --template brand/templates/proposal/src/03-cq-simple.md \
  --project  docs/commercial/project.yaml \
  --var TOTAL_AMOUNT=15000 --var ADVANCE_AMOUNT=7500 --var FINAL_AMOUNT=7500

# Also export PDF
python brand/templates/proposal/src/_assets/export.py \
  --template brand/templates/proposal/src/05-sa-software-field.md \
  --project  docs/commercial/project.yaml --pdf
```

Output: `docs/commercial/[DOCTYPE]-[CODE]-[YEAR]-[NNN]-v1.0.md`
PDFs: `docs/commercial/export/` — **add to `.gitignore`**. Transmit via secure channel.

---

## File naming

```
[DOCTYPE]-[CLIENTCODE]-[YEAR]-[NNN]-v[x.y].md
```

| DOCTYPE | Document |
|---------|----------|
| `NDA` | Non-Disclosure Agreement |
| `TO` | Technical Offer |
| `CQ` | Commercial Quote |
| `SA` | Service Agreement |
| `POI` | Purchase Order Inbound (client → MATESTAIN) |
| `POO` | Purchase Order Outbound (MATESTAIN → supplier) |
| `INV` | Invoice |
| `BOM` | Bill of Materials |

Client code registered in `company/pricing/client-registry.md`.

---

## .gitignore additions for this folder

```
docs/commercial/export/
docs/commercial/project.yaml   # optional — contains client contact details
```
