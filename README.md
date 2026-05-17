# Privacy-Preserving Legal Compliance & RAG Assistant

## Executive Overview

A **Privacy-Preserving Legal RAG (Retrieval-Augmented Generation) Compliance Engine** that scans uploaded legal documents against a structured Indian legal rules database, detects statutory violations with evidence-backed citations, and provides remediation steps — all grounded strictly in verifiable context to eliminate hallucinationsand detect legal documents for legit.

---

## Architecture

```
┌─────────────────────┐       HTTP (REST)       ┌──────────────────────────────┐
│   Next.js Frontend  │ ◄──────────────────────► │   FastAPI Backend            │
│   Port 3000         │                          │   Port 8000                  │
│                     │     POST /upload          │   ┌──────────────────────┐  │
│   - Document Upload │ ──────────────────────►  │   │  PDF Text Extractor   │  │
│   - Analysis View   │                          │   │  (PyMuPDF / fitz)     │  │
│   - Chat Assistant  │     GET /analysis         │   └──────┬───────────────┘  │
│                     │ ◄──────────────────────  │          │                  │
│                     │     POST /ask             │   ┌──────▼───────────────┐  │
│                     │ ──────────────────────►  │   │  FAISS Vector Store   │  │
│                     │                          │   │  (bge-small-en)       │  │
│                     │                          │   └──────┬───────────────┘  │
│                     │                          │          │                  │
│                     │                          │   ┌──────▼───────────────┐  │
│                     │                          │   │  Mistral LLM (API)   │  │
│                     │                          │   │  (JSON mode forced)  │  │
│                     │                          │   └──────────────────────┘  │
└─────────────────────┘                          └──────────────────────────────┘
```

---

## Data Pipeline

### 1. Startup — Rules Ingestion

On server boot, `dataset_loader.py` processes two data sources:

- **`legal_vector_dataset/compliance_rules.json`** — 8 structured document-type compliance rules (Sale Deed, Lease Agreement, Gift Deed, Partnership Deed, MOA/AOA, Employment Contract, E-Agreement, Will). Each rule is broken into 3 semantically focused embedding chunks:
  - Mandatory Clauses (what the document must contain)
  - Critical Violations (what to flag, with legal consequences)
  - Remediation Steps (how to fix each violation)

- **`rules/*.pdf`** — User-provided PDF rule documents. Text is extracted via PyMuPDF, split into 1000-character overlapping chunks, and embedded.

All chunks are embedded using `BAAI/bge-small-en` (384-dimensional dense vectors) and indexed into an in-memory FAISS `IndexFlatL2` store.

### 2. Document Upload & Scan (`POST /upload`)

```
PDF Upload
    │
    ▼
PyMuPDF Text Extraction
    │
    ▼
Document text[:1500] ──► FAISS vector_store.search(k=10)
                                │
                                ▼
                    Filter by SIMILARITY_THRESHOLD (dist < 1.5)
                    (irrelevant rules are dropped before reaching the LLM)
                                │
                                ▼
              build_scan_messages(document_text, filtered_rules)
                                │
                                ▼
              ┌─────────────────────────────────────────────┐
              │  Mistral API (mistral-small-latest)          │
              │  System: Indian Legal Compliance Engine      │
              │  User: Full document text + matched rules    │
              │  response_format: json_object (forced)       │
              │  temperature: 0.2  |  max_tokens: 2048      │
              └────────────────┬────────────────────────────┘
                               │
                               ▼
                    Strict JSON Output:
                    {
                      document_type, confidence, summary,
                      issues: [{severity, title, description,
                                evidence, applicable_law,
                                location, recommendation}],
                      metrics: {critical_issues, warnings,
                                compliant_sections}
                    }
```

### 3. Chat Q&A (`POST /ask`)

```
User Question ──► FAISS search (k=5)
                       │
                       ▼
              Similarity threshold gate
                       │
                       ▼
            build_messages(question, contexts)
                       │
                       ▼
                Mistral API ──► JSON response ──► Frontend chat
```

---

## Anti-Hallucination Controls

| Control | Mechanism |
|---------|-----------|
| Content-driven retrieval | Only the document's own text is used as the search query — no hardcoded generic queries |
| Similarity threshold | Rules with `dist > 1.5` are dropped before reaching the LLM |
| Evidence requirement | Every flagged issue must include an `evidence` field quoting exact text from the document |
| Prompt grounding | System prompt: *"Do NOT invent violations. If the document does not contain enough information to confirm a violation, do NOT flag it."* |
| JSON mode | Mistral's native `response_format={"type": "json_object"}` enforced |
| Low temperature | `temperature=0.2` for deterministic, conservative output |

---

## Strict JSON Adjudication Flow

The inference engine operates under a rigid 7-step pipeline:

1. **Identify document type** (e.g., Sale Deed, Lease, NDA)
2. **Detect missing mandatory elements** with evidence from document text
3. **Detect statutory violations** confirmed by document content
4. **Map violations to Indian Law** (Registration Act 1908, Stamp Act 1899, etc.)
5. **Classify severity** (Critical, High, Medium, Low)
6. **Provide corrective remediation steps**
7. **Output STRICT JSON only** — enforced at the API level

---

## Compliance Rules Coverage

| Document Type | Applicable Laws | Risk Weight |
|---------------|----------------|-------------|
| Sale Deed | Transfer of Property Act 1882, Registration Act 1908, Stamp Act 1899, Contract Act 1872, Income Tax Act 1961 | 40 |
| Lease Agreement | Transfer of Property Act 1882, Registration Act 1908, Stamp Act 1899 | 30 |
| Gift Deed | Transfer of Property Act 1882, Registration Act 1908, Stamp Act 1899 | 35 |
| Partnership Deed | Indian Partnership Act 1932 | 20 |
| MOA / AOA | Companies Act 2013 | 35 |
| Employment Contract | Contract Act 1872, Industrial Disputes Act 1947, Payment of Wages Act 1936 | 25 |
| E-Agreement | IT Act 2000, Contract Act 1872 | 30 |
| Will | Indian Succession Act 1925 | 30 |

---

## Key Files

| File | Role |
|------|------|
| `main.py` | FastAPI app — routes (`/upload`, `/ask`, `/analysis`, `/health`), orchestration |
| `embed_store.py` | `LegalVectorStore` — FAISS index, bge-small-en embeddings, semantic search |
| `dataset_loader.py` | Loads JSON rules + PDF rules into the vector store at startup |
| `prompt_builder.py` | `build_messages()` for chat, `build_scan_messages()` for document scanning |
| `mistral.py` | Mistral API wrapper — structured messages, JSON mode enforcement |
| `compliance_rules.json` | 8 document-type compliance rules (structured JSON) |
| `rules/*.pdf` | User-uploaded PDF rule documents |
| `legalcompalliance/` | Next.js frontend (React, Tailwind CSS, shadcn/ui) |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 16, React, Tailwind CSS, shadcn/ui |
| Backend | FastAPI, Uvicorn, Pydantic |
| Embeddings | sentence-transformers (BAAI/bge-small-en, 384-dim) |
| Vector DB | FAISS IndexFlatL2 (in-memory, CPU) |
| LLM | Mistral AI (mistral-small-latest) via API |
| PDF Parsing | PyMuPDF (fitz) |
| Environment | python-dotenv, .env for API keys |

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/upload` | Upload PDF(s) — extracts text, scans against rules DB, returns compliance JSON |
| `GET` | `/analysis` | Returns the latest scan results (issues, metrics, severity) |
| `POST` | `/ask` | Chat Q&A — retrieves relevant context and generates grounded response |
| `GET` | `/health` | Health check |


anna nee mass naaa
hello
hii
