# Design Spec: Local Procurement KB — Data Pipeline + MCP Server

**Date:** 2026-06-18
**Context:** Caseware "Senior AI Data Software Developer" take-home.
**Goal:** A local data pipeline + MCP server over a procurement/inventory knowledge
base so an MCP-compatible AI agent can retrieve, compare, and reason across documents
and return grounded answers with source citations.

---

## 1. Constraints (from the brief)

- **Time budget:** 3–5 hours total. "Do not overengineer."
- **Local-only:** no cloud, no production infra, no distributed systems, no polished frontend.
- **Reproducible & re-runnable** pipeline.
- **Hard rule:** the MCP layer must consume **pre-processed/indexed** data — no extraction
  or OCR at request time.
- **Must use AI-assisted development** and be able to explain what was AI-assisted and how
  it was validated.

### Locked decisions (agreed with user)

| Decision | Choice |
|---|---|
| Overall approach | **Hybrid** — structured extraction for relational questions + text retrieval for the contract/free-text |
| Language/stack | **Python** |
| Demo MCP client | **MCP Inspector** |
| Execution | Claude builds it after spec+plan approval; user reviews/validates |
| Text retrieval | **BM25 / SQLite FTS5 only** (no embeddings; embeddings listed as future work) |
| Lineage depth | **Lightweight**: `file + doc_type + page + order_id + raw_text` per result (no normalized spans table, no bbox) |

---

## 2. Verified dataset facts (`data/`)

| Folder | Count | Format | Notes |
|---|---|---|---|
| `invoices/` | 8 PDFs + 5 JPGs | text PDF / image | PDFs `invoice_<id>.pdf`; JPGs `batch*.jpg` are image-based (need OCR) |
| `purchase_orders/` | 8 PDFs | text | `purchase_orders_<id>.pdf` |
| `shipping_orders/` | 16 PDFs | text | `order_<id>.pdf` |
| `inventory_reports/` | 7 PDFs | text | `StockReport_<YYYY-MM>_<n>.pdf`, monthly per-category stock tables, 2016-07..2018-01 |
| `contracts/` | 1 PDF | text (narrative) | TotalEnergies master supply contract |

- The PDFs are the **Northwind** dataset; `Order ID: <n>` appears in the text and is the
  **join key** linking invoices ↔ POs ↔ shipping orders. `pdftotext` extracts cleanly.
- The **5 JPG invoices** OCR to a *different* schema (`Invoice no: 23285582`, vendor
  Harris-Green, marble furniture) with **no Order ID** — they do **not** join to the
  Northwind chain. Deliberate "pragmatic handling" test: ingest + isolate, never force-join.
- Installed tooling confirmed: `pdftotext`, `tesseract`, `pdfplumber`.

### Cross-reference matrix (the regression oracle)

This is precomputed from filenames/`Order ID`. The build-time self-check asserts the
rebuilt store reproduces it.

- **Invoices missing a PO:** `10436, 10687, 10839`
- **POs with no invoice:** `10372, 10514, 10657`
- **Fully matched (invoice + PO + shipping):** `10248, 10382, 10442, 10603, 10711`
- **All invoices have a shipment** (no missing shipments).

The brief's example questions ("which invoices are missing a matching PO?", "which
documents support order 10687?") map directly onto this matrix.

---

## 3. Architecture

Two layers with hard separation.

```
            OFFLINE (build.py)                              ONLINE (server.py)
data/ ─► discover → extract → parse → chunk → load → validate ─► kb.sqlite ─► FastMCP (stdio) ─► MCP Inspector
 raw    (route by   (pdftotext/  (regex   (contract  (SQLite+    (self-check)   read-only        5 tools
 files   type)       pdfplumber/  fields)  clauses)   FTS5)       vs matrix)     SQL/FTS only     + 2 resources
                     pytesseract)                                                                 + SourceRef
```

The pipeline does all extraction/OCR/indexing **once** and writes `kb.sqlite`. The server
opens it **read-only** and runs only parameterized SQL/FTS queries.

---

## 4. Data pipeline (`build.py`)

Stage-based, **wipe-and-rebuild**, deterministic.

1. **Discover & hash** — walk `data/`, classify by folder + extension, SHA-256 each file
   into `manifest.json`. The hash caches the slow OCR step so re-runs are instant.
2. **Extract** — route per file:
   - text PDFs → `pdftotext -layout` (column-aligned, regex-friendly)
   - inventory stock tables → `pdfplumber.extract_tables()` (geometry needed for clean rows)
   - contract → `pdftotext` (reflowed prose, no `-layout`)
   - JPGs (and any zero-text PDF) → Pillow preprocess (grayscale, upscale, Otsu) +
     `pytesseract --psm 6`; capture mean word confidence.
   - Routing rule: `.jpg` → OCR unconditionally; `.pdf` → run `pdftotext`, if char yield is
     near-zero treat as image-PDF and OCR (path coded for robustness even though no PDF in
     this dataset triggers it).
3. **Parse fields** — label-anchored regex per doc type. Per-field failure → `null` +
   append to `warnings[]`; the batch always completes. Tag `source_schema`
   (`northwind` | `external_invoice`). Normalize currency/dates at load; keep original
   string as `raw_text` for citation.
4. **Chunk** — clause/heading-aware split of the contract (token cap ~500–800, small
   overlap), char-window fallback if structure detection is weak. Each chunk keeps
   `page` + `section`. Structured docs contribute one raw-text chunk each so search spans
   the whole corpus.
5. **Load** — write SQLite tables + FTS5 index; populate `orders` spine, `order_documents`,
   and precomputed `match_status`.
6. **Validate & report** — assert rebuilt store reproduces the cross-reference matrix
   (§2); emit expected doc counts per type and the unlinked-invoice list; flush
   `warnings.log`. Divergence fails the build loudly.

### Artifacts

```
artifacts/            (git-ignored, fully regenerable)
  raw_text/*.txt      # per-source extraction output (debuggable)
  manifest.json       # SHA-256 hash cache + provenance
  warnings.log        # extraction warnings/skips
  kb.sqlite           # the single store the MCP server reads
```

---

## 5. Data model (SQLite)

```
-- document / provenance layer
documents(doc_id PK, doc_type, source_path, source_format, source_schema,
          page_count, is_joinable, ocr_confidence)

-- order spine + support set
orders(order_id PK)
order_documents(order_id FK, role, doc_id FK, PK(order_id, role, doc_id))

-- per-type structured tables (raw_text kept per key field for citation)
invoices(doc_id PK, order_id FK NULL, vendor, invoice_date, currency, total, raw_text ...)
purchase_orders(doc_id PK, order_id FK NULL, po_date, vendor, total, raw_text ...)
shipping_orders(doc_id PK, order_id FK NULL, ship_date, carrier, ship_to, raw_text ...)

-- unified line items across roles
line_items(line_id PK, doc_id FK, order_id NULL, role, product_id, product_name,
           qty, unit_price, line_total)

-- inventory (no order id; its own dimension)
inventory_stock(inv_id PK, doc_id FK, period, category, product_id, product_name,
                qty_on_hand, unit_price)

-- text retrieval
chunks(chunk_id PK, doc_id FK, page, section, ord, text)
chunks_fts                      -- FTS5 virtual table over chunks.text (bm25 ranking)

-- views
v_order_rollup    -- one row per order; match_status ∈ {matched, missing_po,
                  --   missing_invoice, partial}
v_unmatched_invoices  -- invoices WHERE order_id IS NULL (the image set)
```

**Design notes**
- `orders` is the spine; the `order_id` column on each per-type table *is* the foreign key —
  joins are native SQL.
- `line_items` is one unified table (nullable columns for role-specific fields) instead of
  three near-identical tables.
- `order_documents` materializes the "support set" so "which documents support order X" is a
  single indexed lookup.
- **Isolation invariant:** image invoices get `is_joinable=0`, `order_id=NULL`, and no
  `order_documents` rows → they can never enter the spine, rollup, or any join, so they
  cannot create phantom matches. They remain individually queryable/citable and are surfaced
  via `v_unmatched_invoices`.

### Cross-document questions → SQL

- "Invoices missing a PO" → `v_order_rollup WHERE match_status='missing_po'`
- "POs with no invoice" → `match_status='missing_invoice'`
- "Which documents support order X" → `order_documents` joined to `documents`
- "Does shipment match invoice / mismatches" → compare `line_items` / totals across roles
  on `(order_id, product_id)`.

---

## 6. Retrieval strategy

**Tool-based hybrid routing over one SQLite store.** Two paths, one DB; the agent routes by
choosing the tool (no LLM classifier).

- **Path A — structured:** SQL + metadata filters over extracted fields and the `order_id`
  join. Handles all relational questions (missing-PO, mismatches, "documents supporting
  order X", inventory by period/category). Deterministic — correctness questions where
  semantic search would be actively wrong.
- **Path B — text:** **SQLite FTS5 with `bm25()` ranking** over contract chunks + raw doc
  text. Handles "summarize contract terms…" and "find evidence about vendor/item X".

**Why BM25, not embeddings:** the corpus is tiny (~40 PDFs + 5 JPGs + 1 contract) and the
text questions are keyword-heavy (vendor names, SKUs, "supply of goods"). FTS5 ships inside
the SQLite stdlib — zero model download, fully offline, fully reproducible. An embedding
model download (~90–400 MB) would consume a large share of the time budget and introduce an
offline/reproducibility failure mode the brief warns against. Embeddings would be the right
next step at thousands of docs or for heavily paraphrastic queries → **listed as future
work**.

**Bounding (avoid raw-data dumps):** text path returns top-k (default 5, hard cap ~8/20)
windowed `snippet()` excerpts with scores — never whole docs. Structured path projects only
needed columns (no `SELECT *`), applies `LIMIT`, returns computed verdicts over raw rows.

---

## 7. MCP server (`server.py`, FastMCP / stdio)

Thin, read-only. Opens `kb.sqlite` with `mode=ro`. Each tool maps to a fixed parameterized
read; the only request-time computation is in-memory comparison of already-extracted values
in `reconcile_order`.

### Tools (5)

| Tool | Parameters | Purpose / answers |
|---|---|---|
| `find_orders` | `match_status?` (enum), `vendor?`, `order_id?`, `limit=25`, `offset=0` | discovery + "which invoices are missing a PO?" (`missing_po` → 10436/10687/10839) |
| `get_order_documents` | `order_id`, `doc_type?` | "which documents support order 10687?" |
| `reconcile_order` | `order_id`, `fields?` | "what PO supports this invoice?", "does the shipment match?", "mismatches?" → per-field `match`/`mismatch`/`unavailable` verdict + one-line summary |
| `search_evidence` | `query`, `doc_type?`, `order_id?`, `limit=5` | "summarize contract terms…", "find evidence about vendor/item X" → bounded snippets |
| `list_inventory_reports` | `period?`, `category?`, `limit=25`, `offset=0` | "what inventory reports exist and what period?" |

### Resources (2, thin)

- `kb://catalog` — corpus-level counts (orders, docs per type, contract presence, schema
  version) for instant agent orientation without a tool call.
- `kb://doc/{doc_id}` — open one document's stored text on demand by URI; keeps bulk text out
  of default tool results.

### Question → tool coverage

| Example question | Tool(s) |
|---|---|
| Which invoices are missing a matching PO? | `find_orders(match_status="missing_po")` |
| What PO supports this invoice? | `get_order_documents` → `reconcile_order` |
| Does this shipment match the invoice? | `reconcile_order(order_id)` |
| Summarize contract terms for supply of goods | `search_evidence(query, doc_type="contract")` |
| Which documents support order 10687? | `get_order_documents(10687)` |
| Mismatches between invoice/PO/shipping? | `reconcile_order(order_id)` / `find_orders(match_status=...)` |
| What inventory reports exist and what period? | `list_inventory_reports` |
| Find evidence about vendor/order/item | `search_evidence(query, order_id?)` |

### Citation strategy

Single canonical object on every result (`sources[]` on list/reconcile, per-hit `source` on
search):

```json
{
  "file": "invoices/invoice_10687.pdf",
  "doc_type": "invoice",
  "doc_id": "invoice_10687",
  "order_id": 10687,
  "page": 1,
  "raw_text": "Order ID: 10687"
}
```

`order_id` is `null` for the contract, inventory reports, and image invoices. The agent cites
by `file` (+ `page`) and may open `kb://doc/{doc_id}` to verify before asserting — every claim
traces back to a file/page.

---

## 8. Error / empty-result handling

Absence is a structured answer, not an exception:
- Order has no PO → `documents_missing:["purchase_order"]`, dependent checks `"unavailable"`.
- Unknown `order_id` → `{found:false, message:...}`.
- Empty search → `{count:0, hits:[]}`.
- Out-of-range `limit` → clamped to max.
- **Only** a missing/corrupt `kb.sqlite` returns an MCP `isError` telling the user to run
  `build.py`. Read-only connection makes writes structurally impossible.

---

## 9. Testing / verification

- **Build-time self-check:** rebuilt store must reproduce the cross-reference matrix (§2).
- **Pipeline checks:** expected doc counts per type; every Northwind doc has an `order_id`;
  all 5 image invoices have `order_id=NULL, is_joinable=0`.
- **MCP evidence (Inspector session, screenshots = deliverable):**
  - `find_orders(match_status="missing_po")` → `{10436, 10687, 10839}`
  - `reconcile_order(10248)` → `matched`
  - `reconcile_order(10687)` → `incomplete`, missing PO
  - `search_evidence("supply of goods")` → contract clause snippet with citation

---

## 10. Tech stack & repo

- **Python**, `uv` / `pyproject.toml` with pinned deps.
- Extraction: `pdftotext` (Poppler), `pdfplumber`, `pytesseract`, `Pillow`.
- Store: stdlib `sqlite3` + FTS5. **No** DuckDB / FAISS / Chroma / embeddings.
- MCP: official `mcp` SDK (`FastMCP`), stdio transport.
- Entrypoints: `make build` (run pipeline → `kb.sqlite`), `make inspect` (launch Inspector).

```
.
├── data/                      # provided KB (committed)
├── pipeline/                  # build.py + stage modules
├── server.py                  # FastMCP server
├── artifacts/                 # generated (git-ignored)
├── docs/                      # this spec, architecture & data-flow notes, presentation
├── pyproject.toml
├── Makefile
└── README.md
```

---

## 11. Deliverables coverage (brief checklist)

| Required | Where |
|---|---|
| Source code repo | repo root |
| README + setup | `README.md` |
| Run pipeline instructions | `make build` (README) |
| Run/connect MCP server | `make inspect` (README) |
| Architecture overview | §3 + `docs/` |
| Data-flow overview | §3–§4 |
| Data-modeling explanation | §5 |
| Retrieval-strategy explanation | §6 |
| MCP tools/resources description | §7 |
| Citation strategy | §7 |
| Evidence with MCP client | §9 Inspector screenshots |
| AI-usage explanation | `docs/` AI-usage note |
| Presentation (10–15 min) | `docs/` outline |

---

## 12. Out of scope / future work (deliberate omissions = judgment)

- No vector embeddings / semantic search (BM25 sufficient at this scale; future work).
- No bounding-box lineage / normalized spans table (file+page+raw_text sufficient).
- No DuckDB, FAISS, Chroma, or any extra service.
- No LLM query classifier (tool selection *is* routing).
- No cloud, no auth, no frontend.
