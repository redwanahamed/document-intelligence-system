# document-intelligence-system
# AI-Powered Document Intelligence System

A production-structured platform for processing, extracting, validating, and reviewing invoice documents. Supports PDF (digital and scanned), JPG, PNG, TIFF, XLSX, XLS, and CSV.

---

## Quick Start

```bash
# 1. Start PostgreSQL and Redis
docker compose up -d

# 2. Install backend dependencies and migrate
cd backend && npm install && npm run db:migrate

# 3. Install frontend dependencies
cd ../frontend && npm install && cd ..

# 4. Copy and configure environment
cp backend/.env.example backend/.env
# Edit backend/.env — set OPENAI_API_KEY if you have one;
# the system works without it using a local regex fallback.
echo "NEXT_PUBLIC_API_URL=http://localhost:3001/api/v1" > frontend/.env.local

# 5. Start all three processes (three terminals)
cd backend && npm run dev        # Terminal 1 — API  :3001
cd backend && npm run worker     # Terminal 2 — Worker
cd frontend && npm run dev       # Terminal 3 — UI   :3000
```

Open **http://localhost:3000**.

---

## Full Documentation

| Document | Contents |
|---|---|
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | Component diagram, processing pipeline, adapter interfaces, security controls, AI prompt disclosure, scalability notes, known limitations |
| [`docs/SCHEMA.md`](docs/SCHEMA.md) | Entity-relationship diagram for all 12 tables, validation issue codes, index list |
| [`docs/SETUP.md`](docs/SETUP.md) | Step-by-step setup, environment variable reference, test instructions, evaluation instructions, walkthrough guide |

---

## Supported File Types

| Format | Method |
|---|---|
| PDF (digital/searchable) | Embedded text extraction via `pdf-parse` |
| PDF (scanned) | Per-page image render → `sharp` preprocessing → Tesseract.js OCR |
| JPG · PNG · TIFF | `sharp` preprocessing → Tesseract.js OCR |
| XLSX · XLS | `xlsx` (all worksheets, row/column metadata) |
| CSV | `csv-parse` (encoding detection, delimiter detection, BOM removal) |

---

## REST API

| Method | Endpoint | Purpose |
|---|---|---|
| `POST` | `/api/v1/documents` | Upload one or more files |
| `GET` | `/api/v1/documents` | List documents (filterable, paginated) |
| `GET` | `/api/v1/documents/:id` | Document metadata + job status |
| `GET` | `/api/v1/documents/:id/pages` | Page-level extracted text |
| `GET` | `/api/v1/documents/:id/extraction` | Structured extraction with confidence scores |
| `GET` | `/api/v1/documents/:id/validation` | Errors and warnings |
| `PATCH` | `/api/v1/documents/:id/extraction` | Save manual corrections (audited) |
| `POST` | `/api/v1/documents/:id/reprocess` | Retry failed documents |
| `GET` | `/api/v1/jobs/:id` | Processing progress and failure details |
| `GET` | `/api/v1/search` | Keyword search over document text |
| `POST` | `/api/v1/compare` | PDF-to-PDF or PDF-to-Excel comparison |
| `GET` | `/api/v1/evaluation` | Real-time metrics + latest evaluation run |

---

## Extraction Schema

Every extracted field carries provenance. Missing values are `null`, never invented.

```json
{
  "invoice_number": { "value": "INV-001", "confidence": 0.98, "source_text": "Invoice No: INV-001", "page": 1 },
  "supplier_name":  { "value": null,      "confidence": 0,    "source_text": null,                    "page": null }
}
```

---

## Validation Checks (deterministic, no LLM)

- Required fields present (invoice number, date, supplier, total, currency)
- Date parsing and ordering (due date ≥ invoice date)
- Non-negative amounts
- ISO 4217 currency code
- Line-item math: `qty × unit_price ≈ line_total` (±2%)
- Subtotal = Σ line totals
- `subtotal + tax ≈ total` (±2%)
- Duplicate detection (same invoice number + supplier)
- Low-confidence fields flagged for manual review

---

## Project Structure

```
.
├── backend/
│   ├── src/
│   │   ├── api/
│   │   │   ├── middleware/    error-handler
│   │   │   └── routes/        documents · jobs · search · compare · evaluation
│   │   ├── comparison/        compareDocuments()
│   │   ├── config/            typed config from .env
│   │   ├── db/                pg Pool · migrations · migrate.ts
│   │   ├── evaluation/        run.ts (metrics script)
│   │   ├── extraction/        extractor.ts (OpenAI + local fallback)
│   │   ├── processors/        pdf-text · pdf-ocr · image-ocr · spreadsheet · csv · factory
│   │   ├── queue/             index.ts (BullMQ) · worker.ts
│   │   ├── search/            embeddings.ts (GIN keyword + chunk storage)
│   │   ├── storage/           LocalStorageService
│   │   ├── types/             shared TypeScript types
│   │   ├── utils/             logger · file-validator
│   │   ├── validation/        validator.ts (deterministic rules)
│   │   ├── app.ts             Express app
│   │   └── index.ts           server entry point
│   ├── tests/
│   │   ├── unit/              validator.test · processors.test
│   │   └── integration/       api.test
│   └── evaluation/
│       ├── fixtures/          labeled test documents (populate manually)
│       └── fixtures.py        fixture schema generator
├── frontend/
│   └── src/
│       ├── app/               Next.js App Router pages
│       ├── components/        upload · documents · search · comparison · ui
│       ├── lib/               api.ts (axios client)
│       ├── hooks/             shared React hooks
│       └── types/             frontend type definitions
└── docs/
    ├── ARCHITECTURE.md
    ├── SCHEMA.md
    └── SETUP.md
```

---

## AI and Model Disclosure

- **Extraction model:** OpenAI `gpt-4o` (configurable via `OPENAI_MODEL`)
- **Embeddings model:** `text-embedding-3-small` (configurable via `OPENAI_EMBEDDING_MODEL`)
- **Temperature:** 0 for extraction (deterministic outputs)
- **Anti-hallucination:** Every non-null extracted value must include `source_text` from the document. The extractor nullifies values where source text is missing or cannot be located.
- **Local fallback:** When `OPENAI_API_KEY` is not set or OpenAI fails, a regex-based extractor is used. It is clearly labeled at runtime: `"Using local extraction fallback (not AI-grade accuracy)"`. Confidence scores are capped at 0.5.
- **Code development disclosure:** This codebase was developed with AI coding assistance. All logic, validation rules, security controls, and schema are explicit and independently auditable in source.
