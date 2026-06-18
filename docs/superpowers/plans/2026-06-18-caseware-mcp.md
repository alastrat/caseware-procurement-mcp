# Caseware Procurement MCP — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a local data pipeline that loads a procurement/inventory document set into a single SQLite store, and a read-only MCP server that answers grounded, cited, cross-document questions over it.

**Architecture:** Offline `build.py` extracts (pdftotext/pdfplumber/pytesseract) → parses fields (regex) → chunks the contract → loads `artifacts/kb.sqlite` (tables + FTS5) → self-checks against a known cross-reference matrix. Online `server.py` (FastMCP/stdio) opens the DB read-only and exposes 5 order-centric tools + 2 resources; it runs only parameterized SQL/FTS — no extraction at request time.

**Tech Stack:** Python 3.11+, `uv`/`pyproject.toml`, `pdfplumber`, `pytesseract`, `Pillow`, Poppler `pdftotext` (CLI), stdlib `sqlite3` + FTS5, official `mcp` SDK (`FastMCP`), `pytest`.

## Global Constraints

- **Local-only.** No cloud, no network calls at runtime, no extra services (no DuckDB/FAISS/Chroma/embeddings).
- **MCP reads pre-processed data only.** No PDF/OCR/parse in the request path. Server opens SQLite with `mode=ro`.
- **Reproducible.** `build.py` is wipe-and-rebuild and deterministic; pinned deps in `pyproject.toml`.
- **Text retrieval = BM25 / SQLite FTS5 only.** No embeddings.
- **Lineage = lightweight.** Every result carries `SourceRef = {file, doc_type, doc_id, order_id|null, page|null, raw_text}`. No bbox, no normalized spans table.
- **Dataset root:** `data/`. **Artifacts:** `artifacts/` (git-ignored). **DB path:** `artifacts/kb.sqlite`, overridable via env `KB_DB_PATH`.
- **Cross-reference matrix (regression oracle):** invoices missing PO = {10436, 10687, 10839}; POs with no invoice = {10372, 10514, 10657}; fully matched = {10248, 10382, 10442, 10603, 10711}; all invoices have a shipment.
- **Commit** after each task. End commit messages with: `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`.

---

## File Structure

```
pyproject.toml              # deps + pytest config
Makefile                    # build / inspect / test targets
src/kb/
  __init__.py
  config.py                 # paths, doc-type constants, dataset root
  db.py                     # SQLite connect helpers (rw for build, ro for serve), schema apply
  schema.sql               # DDL: tables, FTS5, views
  extract.py                # text-PDF / pdfplumber-table / OCR routing -> raw text
  parse.py                  # per-doc-type field parsers (regex) -> dataclasses
  chunk.py                  # contract clause chunker + per-doc raw chunks
  load.py                   # write parsed records + chunks into SQLite, populate spine/views
  build.py                  # orchestrator: discover -> extract -> parse -> chunk -> load -> validate
  models.py                 # dataclasses: Document, Invoice, PurchaseOrder, ShippingOrder, InventoryRow, LineItem, Chunk, SourceRef
server.py                   # FastMCP server: 5 tools + 2 resources
tests/
  conftest.py               # builds a real kb.sqlite once per session (uses data/)
  test_extract.py
  test_parse.py
  test_load.py
  test_build_validate.py
  test_server_tools.py
docs/
  architecture.md           # architecture + data-flow overview (deliverable)
  ai-usage.md               # what was AI-assisted + validation (deliverable)
  presentation.md           # 10-15 min presentation outline (deliverable)
```

---

## Document formats (verified from real data — parsers depend on these)

**Invoice PDF** (`pdftotext -layout invoices/invoice_<id>.pdf`):
```
Invoice
Order ID: 10687
Customer ID: HUNGO
Order Date: 2017-09-30
Customer Details:
Contact Name:  Patricia McKenna
...
Product Details:
Product ID   Product Name              Quantity   Unit Price
9            Mishi Kobe Niku           50         97.0
                                       TotalPrice 6201.9
```

**Purchase Order PDF** (`pdftotext -layout`):
```
Purchase Orders
Order ID      Order Date      Customer Name
10248         2016-07-04      Paul Henriot
Products
Product ID:   Product:                    Quantity:   Unit Price:
11            Queso Cabrales              12          14
```

**Shipping Order PDF** (`pdftotext -layout`):
```
Order ID: 10248
Ship Name: ...
Customer ID: ...
Customer Name: ...
Order Date: 2016-07-04
Shipped Date: 2016-07-16
Products:
----------------------------
Product: Queso Cabrales
Quantity: 12
Unit Price: 14.0
Total: 168.0
----------------------------
```

**Inventory Report PDF** (`pdfplumber` / `pdftotext -layout`):
```
Stock Report for 2016-12
Category : Beverages
id category : 1
Product        Units Sold   Units in Stock   Unit Price
Chai           15           39               18
```
(Period is also in filename: `StockReport_<YYYY-MM>_<n>.pdf`.)

**Image invoice JPG** (`pytesseract`): different schema — `Invoice no: 23285582`, `Seller: Harris-Green`, `ITEMS`, no `Order ID`. → `source_schema="external_invoice"`, `order_id=NULL`, `is_joinable=0`.

**Contract PDF**: long narrative legal text → chunk only, no fields.

---

## Task 1: Project scaffold + config + models

**Files:**
- Create: `pyproject.toml`, `Makefile`, `src/kb/__init__.py`, `src/kb/config.py`, `src/kb/models.py`, `tests/__init__.py`
- Test: `tests/test_config.py`

**Interfaces:**
- Produces:
  - `config.DATA_ROOT: Path`, `config.ARTIFACTS_DIR: Path`, `config.DB_PATH: Path` (honors `KB_DB_PATH`), `config.DOC_TYPES: dict[str,str]` mapping folder→doc_type.
  - `config.EXPECTED_MATRIX: dict` with keys `missing_po`, `missing_invoice`, `fully_matched` (sets of int).
  - dataclasses in `models.py`: `SourceRef(file:str, doc_type:str, doc_id:str, order_id:int|None, page:int|None, raw_text:str)`, `Document`, `Invoice`, `PurchaseOrder`, `ShippingOrder`, `InventoryRow`, `LineItem(doc_id, order_id, role, product_id, product_name, qty, unit_price, line_total)`, `Chunk(doc_id, page, section, ord, text)`.

- [ ] **Step 1: Write `pyproject.toml`**

```toml
[project]
name = "caseware-kb"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "pdfplumber>=0.11",
    "pytesseract>=0.3.10",
    "Pillow>=10.0",
    "mcp>=1.2.0",
]

[project.optional-dependencies]
dev = ["pytest>=8.0"]

[tool.pytest.ini_options]
pythonpath = ["src"]
testpaths = ["tests"]

[tool.setuptools.packages.find]
where = ["src"]
```

- [ ] **Step 2: Write the failing test** `tests/test_config.py`

```python
from kb import config, models

def test_paths_resolve():
    assert config.DATA_ROOT.name == "data"
    assert config.DB_PATH.name == "kb.sqlite"

def test_doc_types_cover_all_folders():
    assert set(config.DOC_TYPES.values()) == {
        "invoice", "purchase_order", "shipping_order", "inventory_report", "contract"
    }

def test_matrix_constants():
    assert config.EXPECTED_MATRIX["missing_po"] == {10436, 10687, 10839}
    assert config.EXPECTED_MATRIX["fully_matched"] == {10248, 10382, 10442, 10603, 10711}

def test_sourceref_shape():
    s = models.SourceRef(file="invoices/invoice_10687.pdf", doc_type="invoice",
                         doc_id="invoice_10687", order_id=10687, page=1, raw_text="Order ID: 10687")
    assert s.order_id == 10687
```

- [ ] **Step 3: Run test to verify it fails**

Run: `pytest tests/test_config.py -v`
Expected: FAIL (`ModuleNotFoundError: kb`)

- [ ] **Step 4: Write `src/kb/config.py`**

```python
import os
from pathlib import Path

ROOT = Path(__file__).resolve().parents[2]
DATA_ROOT = ROOT / "data"
ARTIFACTS_DIR = ROOT / "artifacts"
DB_PATH = Path(os.environ.get("KB_DB_PATH", ARTIFACTS_DIR / "kb.sqlite"))

DOC_TYPES = {
    "invoices": "invoice",
    "purchase_orders": "purchase_order",
    "shipping_orders": "shipping_order",
    "inventory_reports": "inventory_report",
    "contracts": "contract",
}

EXPECTED_MATRIX = {
    "missing_po": {10436, 10687, 10839},
    "missing_invoice": {10372, 10514, 10657},
    "fully_matched": {10248, 10382, 10442, 10603, 10711},
}
```

- [ ] **Step 5: Write `src/kb/models.py`**

```python
from dataclasses import dataclass, field

@dataclass
class SourceRef:
    file: str
    doc_type: str
    doc_id: str
    order_id: int | None
    page: int | None
    raw_text: str

@dataclass
class LineItem:
    doc_id: str
    order_id: int | None
    role: str
    product_id: int | None
    product_name: str
    qty: float | None
    unit_price: float | None
    line_total: float | None

@dataclass
class Chunk:
    doc_id: str
    page: int | None
    section: str | None
    ord: int
    text: str

@dataclass
class Document:
    doc_id: str
    doc_type: str
    source_path: str
    source_format: str          # pdf | image
    source_schema: str          # northwind | external_invoice | contract | inventory
    page_count: int
    is_joinable: bool
    ocr_confidence: float | None = None

@dataclass
class Invoice:
    doc_id: str
    order_id: int | None
    customer_id: str | None
    contact_name: str | None
    order_date: str | None
    total: float | None
    raw_text: str
    line_items: list[LineItem] = field(default_factory=list)

@dataclass
class PurchaseOrder:
    doc_id: str
    order_id: int | None
    order_date: str | None
    customer_name: str | None
    total: float | None
    raw_text: str
    line_items: list[LineItem] = field(default_factory=list)

@dataclass
class ShippingOrder:
    doc_id: str
    order_id: int | None
    order_date: str | None
    shipped_date: str | None
    ship_name: str | None
    raw_text: str
    line_items: list[LineItem] = field(default_factory=list)

@dataclass
class InventoryRow:
    doc_id: str
    period: str
    category: str
    product_name: str
    units_sold: float | None
    units_in_stock: float | None
    unit_price: float | None
```

- [ ] **Step 6: Write `src/kb/__init__.py`** (empty) and `tests/__init__.py` (empty), and `Makefile`

```makefile
.PHONY: build inspect test
build:
	python -m kb.build
inspect:
	npx @modelcontextprotocol/inspector python server.py
test:
	pytest -q
```

- [ ] **Step 7: Run test to verify it passes**

Run: `pytest tests/test_config.py -v`
Expected: PASS (4 tests)

- [ ] **Step 8: Commit**

```bash
git add pyproject.toml Makefile src tests
git commit -m "feat: project scaffold, config, and data models

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: SQLite schema + DB helpers

**Files:**
- Create: `src/kb/schema.sql`, `src/kb/db.py`
- Test: `tests/test_db.py`

**Interfaces:**
- Consumes: `config.DB_PATH`.
- Produces:
  - `db.connect(path, *, readonly=False) -> sqlite3.Connection` (readonly uses `file:...?mode=ro` URI; sets `row_factory = sqlite3.Row`).
  - `db.init_schema(conn) -> None` (executes `schema.sql`).
  - Tables/views per spec §5: `documents, orders, order_documents, invoices, purchase_orders, shipping_orders, line_items, inventory_stock, chunks, chunks_fts, v_order_rollup, v_unmatched_invoices`.

- [ ] **Step 1: Write `src/kb/schema.sql`**

```sql
PRAGMA foreign_keys = ON;

CREATE TABLE documents (
  doc_id TEXT PRIMARY KEY,
  doc_type TEXT NOT NULL,
  source_path TEXT NOT NULL,
  source_format TEXT NOT NULL,
  source_schema TEXT NOT NULL,
  page_count INTEGER,
  is_joinable INTEGER NOT NULL DEFAULT 1,
  ocr_confidence REAL
);

CREATE TABLE orders (order_id INTEGER PRIMARY KEY);

CREATE TABLE order_documents (
  order_id INTEGER NOT NULL REFERENCES orders(order_id),
  role TEXT NOT NULL,                 -- invoice | purchase_order | shipping_order
  doc_id TEXT NOT NULL REFERENCES documents(doc_id),
  PRIMARY KEY (order_id, role, doc_id)
);

CREATE TABLE invoices (
  doc_id TEXT PRIMARY KEY REFERENCES documents(doc_id),
  order_id INTEGER REFERENCES orders(order_id),
  customer_id TEXT, contact_name TEXT, order_date TEXT, total REAL, raw_text TEXT
);
CREATE TABLE purchase_orders (
  doc_id TEXT PRIMARY KEY REFERENCES documents(doc_id),
  order_id INTEGER REFERENCES orders(order_id),
  order_date TEXT, customer_name TEXT, total REAL, raw_text TEXT
);
CREATE TABLE shipping_orders (
  doc_id TEXT PRIMARY KEY REFERENCES documents(doc_id),
  order_id INTEGER REFERENCES orders(order_id),
  order_date TEXT, shipped_date TEXT, ship_name TEXT, raw_text TEXT
);

CREATE TABLE line_items (
  line_id INTEGER PRIMARY KEY AUTOINCREMENT,
  doc_id TEXT NOT NULL REFERENCES documents(doc_id),
  order_id INTEGER,
  role TEXT NOT NULL,
  product_id INTEGER, product_name TEXT,
  qty REAL, unit_price REAL, line_total REAL
);

CREATE TABLE inventory_stock (
  inv_id INTEGER PRIMARY KEY AUTOINCREMENT,
  doc_id TEXT NOT NULL REFERENCES documents(doc_id),
  period TEXT, category TEXT, product_name TEXT,
  units_sold REAL, units_in_stock REAL, unit_price REAL
);

CREATE TABLE chunks (
  chunk_id INTEGER PRIMARY KEY AUTOINCREMENT,
  doc_id TEXT NOT NULL REFERENCES documents(doc_id),
  page INTEGER, section TEXT, ord INTEGER, text TEXT
);
CREATE VIRTUAL TABLE chunks_fts USING fts5(
  text, doc_id UNINDEXED, chunk_id UNINDEXED, tokenize='porter'
);

CREATE VIEW v_order_rollup AS
SELECT o.order_id,
  MAX(od.role='invoice')         AS has_invoice,
  MAX(od.role='purchase_order')  AS has_po,
  MAX(od.role='shipping_order')  AS has_ship,
  CASE
    WHEN MAX(od.role='invoice') AND MAX(od.role='purchase_order') AND MAX(od.role='shipping_order') THEN 'matched'
    WHEN MAX(od.role='invoice')=1 AND MAX(od.role='purchase_order')=0 THEN 'missing_po'
    WHEN MAX(od.role='purchase_order')=1 AND MAX(od.role='invoice')=0 THEN 'missing_invoice'
    ELSE 'partial'
  END AS match_status
FROM orders o
LEFT JOIN order_documents od USING(order_id)
GROUP BY o.order_id;

CREATE VIEW v_unmatched_invoices AS
SELECT d.doc_id, d.source_path, i.customer_id, i.total
FROM invoices i JOIN documents d USING(doc_id)
WHERE i.order_id IS NULL;
```

- [ ] **Step 2: Write the failing test** `tests/test_db.py`

```python
import sqlite3, pytest
from kb import db

def test_init_schema_creates_tables(tmp_path):
    conn = db.connect(tmp_path / "t.sqlite")
    db.init_schema(conn)
    names = {r["name"] for r in conn.execute(
        "SELECT name FROM sqlite_master WHERE type IN ('table','view')")}
    assert {"documents","orders","order_documents","invoices","purchase_orders",
            "shipping_orders","line_items","inventory_stock","chunks",
            "v_order_rollup","v_unmatched_invoices"} <= names

def test_readonly_blocks_writes(tmp_path):
    p = tmp_path / "t.sqlite"
    db.init_schema(db.connect(p))
    ro = db.connect(p, readonly=True)
    with pytest.raises(sqlite3.OperationalError):
        ro.execute("INSERT INTO orders(order_id) VALUES (1)")
```

- [ ] **Step 3: Run test to verify it fails**

Run: `pytest tests/test_db.py -v`
Expected: FAIL (`ModuleNotFoundError` / `AttributeError`)

- [ ] **Step 4: Write `src/kb/db.py`**

```python
import sqlite3
from pathlib import Path

SCHEMA = Path(__file__).with_name("schema.sql")

def connect(path, *, readonly=False) -> sqlite3.Connection:
    path = Path(path)
    if readonly:
        conn = sqlite3.connect(f"file:{path}?mode=ro", uri=True)
    else:
        path.parent.mkdir(parents=True, exist_ok=True)
        conn = sqlite3.connect(path)
    conn.row_factory = sqlite3.Row
    conn.execute("PRAGMA foreign_keys = ON")
    return conn

def init_schema(conn: sqlite3.Connection) -> None:
    conn.executescript(SCHEMA.read_text())
    conn.commit()
```

- [ ] **Step 5: Run test to verify it passes**

Run: `pytest tests/test_db.py -v`
Expected: PASS (2 tests)

- [ ] **Step 6: Commit**

```bash
git add src/kb/schema.sql src/kb/db.py tests/test_db.py
git commit -m "feat: sqlite schema, views, and read-only connection helper

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: Extraction layer (text PDF / tables / OCR routing)

**Files:**
- Create: `src/kb/extract.py`
- Test: `tests/test_extract.py`

**Interfaces:**
- Consumes: source file paths.
- Produces:
  - `extract.pdf_text(path) -> tuple[str, int]` — returns (`pdftotext -layout` output, page_count). Uses subprocess `pdftotext -layout <path> -`; page count via `-` blank-form-feed split or `pdfinfo`.
  - `extract.pdf_text_reflow(path) -> str` — `pdftotext` without `-layout` (for the contract).
  - `extract.ocr_image(path) -> tuple[str, float]` — returns (text, mean_confidence 0..100) via Pillow preprocess + `pytesseract`.
  - `extract.is_text_pdf(path) -> bool` — True if `pdf_text` yields > 100 non-whitespace chars.

- [ ] **Step 1: Write the failing test** `tests/test_extract.py`

```python
from kb import extract
from kb import config

INV = config.DATA_ROOT / "invoices" / "invoice_10687.pdf"
JPG = config.DATA_ROOT / "invoices" / "batch1-1486.jpg"

def test_pdf_text_extracts_order_id():
    text, pages = extract.pdf_text(INV)
    assert "Order ID: 10687" in text
    assert pages >= 1

def test_is_text_pdf_true_for_invoice():
    assert extract.is_text_pdf(INV) is True

def test_ocr_image_returns_text_and_confidence():
    text, conf = extract.ocr_image(JPG)
    assert "Invoice" in text or "Inv" in text
    assert 0.0 <= conf <= 100.0
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_extract.py -v`
Expected: FAIL (`ModuleNotFoundError`)

- [ ] **Step 3: Write `src/kb/extract.py`**

```python
import subprocess
from pathlib import Path
from PIL import Image, ImageOps
import pytesseract

def pdf_text(path) -> tuple[str, int]:
    out = subprocess.run(["pdftotext", "-layout", str(path), "-"],
                         capture_output=True, text=True, check=True).stdout
    pages = out.count("\f") + 1 if out else 0
    return out, pages

def pdf_text_reflow(path) -> str:
    return subprocess.run(["pdftotext", str(path), "-"],
                          capture_output=True, text=True, check=True).stdout

def is_text_pdf(path) -> bool:
    text, _ = pdf_text(path)
    return len("".join(text.split())) > 100

def ocr_image(path) -> tuple[str, float]:
    img = Image.open(path).convert("L")
    img = ImageOps.autocontrast(img)
    if img.width < 1500:
        scale = 1500 / img.width
        img = img.resize((int(img.width * scale), int(img.height * scale)))
    data = pytesseract.image_to_data(img, config="--psm 6",
                                     output_type=pytesseract.Output.DICT)
    words = [w for w in data["text"] if w.strip()]
    confs = [int(c) for c in data["conf"] if c not in ("-1", -1)]
    text = " ".join(words)
    mean_conf = (sum(confs) / len(confs)) if confs else 0.0
    return text, mean_conf
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_extract.py -v`
Expected: PASS (3 tests). If `tesseract` missing, install via `brew install tesseract`.

- [ ] **Step 5: Commit**

```bash
git add src/kb/extract.py tests/test_extract.py
git commit -m "feat: extraction layer (pdftotext, pdfplumber-ready, OCR routing)

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 4: Field parsers (invoice, PO, shipping, inventory, image invoice)

**Files:**
- Create: `src/kb/parse.py`
- Test: `tests/test_parse.py`

**Interfaces:**
- Consumes: `extract` output strings; `models` dataclasses.
- Produces:
  - `parse.parse_invoice(doc_id, text) -> Invoice`
  - `parse.parse_purchase_order(doc_id, text) -> PurchaseOrder`
  - `parse.parse_shipping_order(doc_id, text) -> ShippingOrder`
  - `parse.parse_inventory(doc_id, text, filename) -> list[InventoryRow]`
  - `parse.parse_external_invoice(doc_id, text) -> Invoice` (order_id=None)
  - Helper `parse.find_order_id(text) -> int | None` (matches `Order ID:` and the PO table row).

- [ ] **Step 1: Write the failing test** `tests/test_parse.py`

```python
from kb import parse, extract, config

def _text(rel):
    return extract.pdf_text(config.DATA_ROOT / rel)[0]

def test_parse_invoice_fields_and_lines():
    inv = parse.parse_invoice("invoice_10687", _text("invoices/invoice_10687.pdf"))
    assert inv.order_id == 10687
    assert inv.total and abs(inv.total - 6201.9) < 0.01
    assert any(li.product_name.startswith("Mishi Kobe") for li in inv.line_items)

def test_parse_po_table_row():
    po = parse.parse_purchase_order("purchase_orders_10248", _text("purchase_orders/purchase_orders_10248.pdf"))
    assert po.order_id == 10248
    assert po.customer_name == "Paul Henriot"
    assert any(li.product_name.startswith("Queso Cabrales") for li in po.line_items)

def test_parse_shipping_products():
    so = parse.parse_shipping_order("order_10248", _text("shipping_orders/order_10248.pdf"))
    assert so.order_id == 10248
    assert any(abs((li.line_total or 0) - 168.0) < 0.01 for li in so.line_items)

def test_parse_inventory_rows():
    rows = parse.parse_inventory("StockReport_2016-12_1",
        _text("inventory_reports/StockReport_2016-12_1.pdf"), "StockReport_2016-12_1.pdf")
    assert rows[0].period == "2016-12"
    assert rows[0].category == "Beverages"
    assert any(r.product_name == "Chai" for r in rows)

def test_find_order_id_none_for_external():
    assert parse.find_order_id("Invoice no: 23285582\nSeller: Harris-Green") is None
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_parse.py -v`
Expected: FAIL (`ModuleNotFoundError`)

- [ ] **Step 3: Write `src/kb/parse.py`**

```python
import re
from kb.models import Invoice, PurchaseOrder, ShippingOrder, InventoryRow, LineItem

def _num(s):
    s = (s or "").replace(",", "").replace("$", "").strip()
    try:
        return float(s)
    except ValueError:
        return None

def find_order_id(text) -> int | None:
    m = re.search(r"Order ID:\s*(\d+)", text)
    if m:
        return int(m.group(1))
    # PO table: a standalone line "10248   2016-07-04   Paul Henriot"
    m = re.search(r"^\s*(\d{4,6})\s+\d{4}-\d{2}-\d{2}\s+\S", text, re.MULTILINE)
    return int(m.group(1)) if m else None

def _label(text, label):
    m = re.search(rf"{re.escape(label)}\s*:?\s*(.+)", text)
    return m.group(1).strip() if m else None

def parse_invoice(doc_id, text) -> Invoice:
    oid = find_order_id(text)
    total = None
    mt = re.search(r"TotalPrice\s+([\d.,]+)", text)
    if mt:
        total = _num(mt.group(1))
    lines = []
    # rows: "<pid>   <name...>   <qty>   <unit_price>"
    for m in re.finditer(r"^\s*(\d+)\s+(.+?)\s{2,}(\d+)\s+([\d.]+)\s*$", text, re.MULTILINE):
        pid, name, qty, up = m.groups()
        if "TotalPrice" in name:
            continue
        lines.append(LineItem(doc_id, oid, "invoice", int(pid), name.strip(),
                              _num(qty), _num(up), None))
    return Invoice(doc_id, oid, _label(text, "Customer ID"), _label(text, "Contact Name"),
                   _label(text, "Order Date"), total, text, lines)

def parse_purchase_order(doc_id, text) -> PurchaseOrder:
    oid = find_order_id(text)
    cust = None
    mrow = re.search(r"^\s*\d{4,6}\s+\d{4}-\d{2}-\d{2}\s+(.+?)\s*$", text, re.MULTILINE)
    if mrow:
        cust = mrow.group(1).strip()
    mdate = re.search(r"(\d{4}-\d{2}-\d{2})", text)
    lines = []
    for m in re.finditer(r"^\s*(\d+)\s+(.+?)\s{2,}(\d+)\s+([\d.]+)\s*$", text, re.MULTILINE):
        pid, name, qty, up = m.groups()
        lines.append(LineItem(doc_id, oid, "purchase_order", int(pid), name.strip(),
                              _num(qty), _num(up), None))
    total = sum((li.qty or 0) * (li.unit_price or 0) for li in lines) or None
    return PurchaseOrder(doc_id, oid, mdate.group(1) if mdate else None, cust, total, text, lines)

def parse_shipping_order(doc_id, text) -> ShippingOrder:
    oid = find_order_id(text)
    lines = []
    for blk in re.split(r"-{5,}", text):
        name = _label(blk, "Product")
        if not name:
            continue
        lines.append(LineItem(doc_id, oid, "shipping_order", None, name,
                              _num(_label(blk, "Quantity")), _num(_label(blk, "Unit Price")),
                              _num(_label(blk, "Total"))))
    return ShippingOrder(doc_id, oid, _label(text, "Order Date"),
                         _label(text, "Shipped Date"), _label(text, "Ship Name"), text, lines)

def parse_inventory(doc_id, text, filename) -> list[InventoryRow]:
    mper = re.search(r"StockReport_(\d{4}-\d{2})", filename) or re.search(r"Stock Report for (\d{4}-\d{2})", text)
    period = mper.group(1) if mper else None
    category = _label(text, "Category")
    rows = []
    for m in re.finditer(r"^\s*(.+?)\s{2,}(\d+)\s+(\d+)\s+([\d.]+)\s*$", text, re.MULTILINE):
        name, sold, stock, price = m.groups()
        if name.strip().lower().startswith("product"):
            continue
        rows.append(InventoryRow(doc_id, period, category, name.strip(),
                                 _num(sold), _num(stock), _num(price)))
    return rows

def parse_external_invoice(doc_id, text) -> Invoice:
    mno = re.search(r"Invoice no:\s*(\S+)", text)
    cust = mno.group(1) if mno else None
    return Invoice(doc_id, None, cust, None, None, None, text, [])
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_parse.py -v`
Expected: PASS (5 tests). If a regex misses on real text, adjust the pattern against `pdftotext -layout` output and re-run — do not change the assertions.

- [ ] **Step 5: Commit**

```bash
git add src/kb/parse.py tests/test_parse.py
git commit -m "feat: per-doc-type field parsers with graceful nulls

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 5: Contract chunker

**Files:**
- Create: `src/kb/chunk.py`
- Test: `tests/test_chunk.py`

**Interfaces:**
- Consumes: text strings.
- Produces:
  - `chunk.chunk_contract(doc_id, text) -> list[Chunk]` — split on headings (`^\d+(\.\d+)*\s`, `ARTICLE`, `PART`, ALL-CAPS lines); cap each chunk to ~1200 chars with ~150-char overlap via a char-window fallback; `section` = nearest preceding heading; `ord` increments.
  - `chunk.chunk_document(doc_id, text, max_chars=1200) -> list[Chunk]` — generic one-or-few raw chunks for non-contract docs (section=None).

- [ ] **Step 1: Write the failing test** `tests/test_chunk.py`

```python
from kb import chunk

def test_chunk_document_bounds_size():
    text = "word " * 1000
    chunks = chunk.chunk_document("d1", text, max_chars=1200)
    assert all(len(c.text) <= 1200 for c in chunks)
    assert chunks[0].ord == 0

def test_chunk_contract_sets_sections():
    text = ("MASTER CONTRACT\n"
            "1. DEFINITIONS\nThe terms below apply.\n"
            "2. SUPPLY OF GOODS\nSupplier shall deliver the Goods within 30 days.\n")
    chunks = chunk.chunk_contract("contract", text)
    assert any("Supplier shall deliver" in c.text for c in chunks)
    assert any(c.section and "SUPPLY OF GOODS" in c.section for c in chunks)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_chunk.py -v`
Expected: FAIL (`ModuleNotFoundError`)

- [ ] **Step 3: Write `src/kb/chunk.py`**

```python
import re
from kb.models import Chunk

HEADING = re.compile(r"^\s*(?:\d+(?:\.\d+)*\.?\s+\S|ARTICLE\b|PART\b|[A-Z][A-Z \-]{6,})", re.MULTILINE)

def _window(text, max_chars, overlap=150):
    out, i = [], 0
    while i < len(text):
        out.append(text[i:i + max_chars])
        i += max_chars - overlap
    return out or [""]

def chunk_document(doc_id, text, max_chars=1200) -> list[Chunk]:
    text = text.strip()
    return [Chunk(doc_id, None, None, n, w)
            for n, w in enumerate(_window(text, max_chars))]

def chunk_contract(doc_id, text) -> list[Chunk]:
    matches = list(HEADING.finditer(text))
    chunks, ordn = [], 0
    if not matches:
        return chunk_document(doc_id, text)
    bounds = [m.start() for m in matches] + [len(text)]
    for idx in range(len(matches)):
        seg = text[bounds[idx]:bounds[idx + 1]].strip()
        section = seg.splitlines()[0].strip()[:120] if seg else None
        for w in _window(seg, 1200):
            chunks.append(Chunk(doc_id, None, section, ordn, w))
            ordn += 1
    return chunks
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_chunk.py -v`
Expected: PASS (2 tests)

- [ ] **Step 5: Commit**

```bash
git add src/kb/chunk.py tests/test_chunk.py
git commit -m "feat: heading-aware contract chunker + generic doc chunker

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 6: Loader (write records, spine, FTS, views)

**Files:**
- Create: `src/kb/load.py`
- Test: `tests/test_load.py`

**Interfaces:**
- Consumes: `db`, `models`.
- Produces:
  - `load.insert_document(conn, doc: Document) -> None`
  - `load.insert_invoice / insert_purchase_order / insert_shipping_order(conn, rec) -> None` (also inserts its `line_items` and, when `order_id` is not None, an `order_documents` row + ensures `orders` row).
  - `load.insert_inventory(conn, rows: list[InventoryRow]) -> None`
  - `load.insert_chunks(conn, chunks: list[Chunk]) -> None` (writes `chunks` + mirrors into `chunks_fts`).
  - `load.ensure_order(conn, order_id) -> None`.

- [ ] **Step 1: Write the failing test** `tests/test_load.py`

```python
from kb import db, load
from kb.models import Document, Invoice, LineItem, Chunk

def _conn(tmp_path):
    c = db.connect(tmp_path / "t.sqlite"); db.init_schema(c); return c

def test_insert_invoice_populates_spine(tmp_path):
    c = _conn(tmp_path)
    load.insert_document(c, Document("invoice_1","invoice","invoices/x.pdf","pdf","northwind",1,True))
    inv = Invoice("invoice_1", 10687, "HUNGO", "Pat", "2017-09-30", 6201.9, "Order ID: 10687",
                  [LineItem("invoice_1", 10687, "invoice", 9, "Mishi", 50, 97.0, None)])
    load.insert_invoice(c, inv)
    assert c.execute("SELECT match_status FROM v_order_rollup WHERE order_id=10687").fetchone()[0] == "missing_po"
    assert c.execute("SELECT COUNT(*) FROM line_items").fetchone()[0] == 1

def test_insert_chunks_searchable(tmp_path):
    c = _conn(tmp_path)
    load.insert_document(c, Document("contract","contract","contracts/c.pdf","pdf","contract",10,False))
    load.insert_chunks(c, [Chunk("contract", None, "2. SUPPLY", 0, "Supplier shall deliver the Goods")])
    hit = c.execute("SELECT text FROM chunks_fts WHERE chunks_fts MATCH 'deliver'").fetchone()
    assert "deliver" in hit["text"]
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_load.py -v`
Expected: FAIL (`ModuleNotFoundError`)

- [ ] **Step 3: Write `src/kb/load.py`**

```python
from kb.models import Document, Chunk

def insert_document(conn, doc: Document):
    conn.execute(
        "INSERT INTO documents VALUES (?,?,?,?,?,?,?,?)",
        (doc.doc_id, doc.doc_type, doc.source_path, doc.source_format,
         doc.source_schema, doc.page_count, int(doc.is_joinable), doc.ocr_confidence))

def ensure_order(conn, order_id):
    if order_id is not None:
        conn.execute("INSERT OR IGNORE INTO orders(order_id) VALUES (?)", (order_id,))

def _link(conn, order_id, role, doc_id):
    if order_id is not None:
        ensure_order(conn, order_id)
        conn.execute("INSERT OR IGNORE INTO order_documents VALUES (?,?,?)",
                     (order_id, role, doc_id))

def _lines(conn, items):
    for li in items:
        conn.execute(
            "INSERT INTO line_items(doc_id,order_id,role,product_id,product_name,qty,unit_price,line_total)"
            " VALUES (?,?,?,?,?,?,?,?)",
            (li.doc_id, li.order_id, li.role, li.product_id, li.product_name,
             li.qty, li.unit_price, li.line_total))

def insert_invoice(conn, inv):
    conn.execute("INSERT INTO invoices VALUES (?,?,?,?,?,?,?)",
                 (inv.doc_id, inv.order_id, inv.customer_id, inv.contact_name,
                  inv.order_date, inv.total, inv.raw_text))
    _link(conn, inv.order_id, "invoice", inv.doc_id)
    _lines(conn, inv.line_items)

def insert_purchase_order(conn, po):
    conn.execute("INSERT INTO purchase_orders VALUES (?,?,?,?,?,?)",
                 (po.doc_id, po.order_id, po.order_date, po.customer_name, po.total, po.raw_text))
    _link(conn, po.order_id, "purchase_order", po.doc_id)
    _lines(conn, po.line_items)

def insert_shipping_order(conn, so):
    conn.execute("INSERT INTO shipping_orders VALUES (?,?,?,?,?,?)",
                 (so.doc_id, so.order_id, so.order_date, so.shipped_date, so.ship_name, so.raw_text))
    _link(conn, so.order_id, "shipping_order", so.doc_id)
    _lines(conn, so.line_items)

def insert_inventory(conn, rows):
    for r in rows:
        conn.execute(
            "INSERT INTO inventory_stock(doc_id,period,category,product_name,units_sold,units_in_stock,unit_price)"
            " VALUES (?,?,?,?,?,?,?)",
            (r.doc_id, r.period, r.category, r.product_name, r.units_sold, r.units_in_stock, r.unit_price))

def insert_chunks(conn, chunks):
    for ch in chunks:
        cur = conn.execute("INSERT INTO chunks(doc_id,page,section,ord,text) VALUES (?,?,?,?,?)",
                           (ch.doc_id, ch.page, ch.section, ch.ord, ch.text))
        conn.execute("INSERT INTO chunks_fts(text,doc_id,chunk_id) VALUES (?,?,?)",
                     (ch.text, ch.doc_id, cur.lastrowid))
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_load.py -v`
Expected: PASS (2 tests)

- [ ] **Step 5: Commit**

```bash
git add src/kb/load.py tests/test_load.py
git commit -m "feat: SQLite loader with spine linking and FTS mirroring

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 7: Build orchestrator + matrix self-check

**Files:**
- Create: `src/kb/build.py`
- Test: `tests/test_build_validate.py`, `tests/conftest.py`

**Interfaces:**
- Consumes: everything above.
- Produces:
  - `build.run(db_path=config.DB_PATH) -> dict` — wipes DB, applies schema, walks `data/`, routes each file (text PDF→parse; jpg→OCR→`parse_external_invoice`; inventory→`parse_inventory`; contract→`chunk_contract`), loads, commits, runs `validate`, returns a stats dict.
  - `build.validate(conn) -> dict` — computes actual missing_po / missing_invoice / fully_matched sets from `v_order_rollup`, asserts equality with `config.EXPECTED_MATRIX`, raises `AssertionError` on mismatch.
  - `python -m kb.build` entrypoint prints stats.
  - `conftest.py` fixture `kb_db` (session-scoped) → builds once into a tmp path, yields the path.

- [ ] **Step 1: Write `tests/conftest.py`**

```python
import pytest
from kb import build

@pytest.fixture(scope="session")
def kb_db(tmp_path_factory):
    path = tmp_path_factory.mktemp("kb") / "kb.sqlite"
    build.run(db_path=path)
    return path
```

- [ ] **Step 2: Write the failing test** `tests/test_build_validate.py`

```python
from kb import db, build

def test_build_reproduces_matrix(kb_db):
    conn = db.connect(kb_db, readonly=True)
    rollup = {r["order_id"]: r["match_status"]
              for r in conn.execute("SELECT order_id, match_status FROM v_order_rollup")}
    missing_po = {oid for oid, s in rollup.items() if s == "missing_po"}
    assert missing_po == {10436, 10687, 10839}

def test_image_invoices_unlinked(kb_db):
    conn = db.connect(kb_db, readonly=True)
    n = conn.execute("SELECT COUNT(*) FROM v_unmatched_invoices").fetchone()[0]
    assert n == 5

def test_doc_counts(kb_db):
    conn = db.connect(kb_db, readonly=True)
    counts = {r["doc_type"]: r["c"] for r in conn.execute(
        "SELECT doc_type, COUNT(*) c FROM documents GROUP BY doc_type")}
    assert counts["purchase_order"] == 8
    assert counts["shipping_order"] == 16
    assert counts["invoice"] == 13   # 8 pdf + 5 jpg
```

- [ ] **Step 3: Run test to verify it fails**

Run: `pytest tests/test_build_validate.py -v`
Expected: FAIL (`ModuleNotFoundError: kb.build`)

- [ ] **Step 4: Write `src/kb/build.py`**

```python
import sys
from pathlib import Path
from kb import config, db, extract, parse, chunk, load
from kb.models import Document

def _doc_id(path: Path) -> str:
    return path.stem

def run(db_path=config.DB_PATH) -> dict:
    db_path = Path(db_path)
    if db_path.exists():
        db_path.unlink()
    conn = db.connect(db_path)
    db.init_schema(conn)
    stats = {"invoice": 0, "purchase_order": 0, "shipping_order": 0,
             "inventory_report": 0, "contract": 0}

    for folder, doc_type in config.DOC_TYPES.items():
        for path in sorted((config.DATA_ROOT / folder).glob("*")):
            if path.name.startswith("."):
                continue
            doc_id = _doc_id(path)
            if path.suffix.lower() == ".jpg":
                text, conf = extract.ocr_image(path)
                load.insert_document(conn, Document(doc_id, "invoice",
                    f"{folder}/{path.name}", "image", "external_invoice", 1, False, conf))
                load.insert_invoice(conn, parse.parse_external_invoice(doc_id, text))
                load.insert_chunks(conn, chunk.chunk_document(doc_id, text))
                stats["invoice"] += 1
                continue

            if doc_type == "contract":
                text = extract.pdf_text_reflow(path)
                _, pages = extract.pdf_text(path)
                load.insert_document(conn, Document(doc_id, "contract",
                    f"{folder}/{path.name}", "pdf", "contract", pages, False))
                load.insert_chunks(conn, chunk.chunk_contract(doc_id, text))
                stats["contract"] += 1
                continue

            text, pages = extract.pdf_text(path)
            schema = "northwind"
            load.insert_document(conn, Document(doc_id, doc_type,
                f"{folder}/{path.name}", "pdf", schema, pages, doc_type != "inventory_report"))
            if doc_type == "invoice":
                load.insert_invoice(conn, parse.parse_invoice(doc_id, text))
            elif doc_type == "purchase_order":
                load.insert_purchase_order(conn, parse.parse_purchase_order(doc_id, text))
            elif doc_type == "shipping_order":
                load.insert_shipping_order(conn, parse.parse_shipping_order(doc_id, text))
            elif doc_type == "inventory_report":
                load.insert_inventory(conn, parse.parse_inventory(doc_id, text, path.name))
            load.insert_chunks(conn, chunk.chunk_document(doc_id, text))
            stats[doc_type] += 1

    conn.commit()
    stats["validation"] = validate(conn)
    conn.commit()
    return stats

def validate(conn) -> dict:
    rollup = {r["order_id"]: r["match_status"]
              for r in conn.execute("SELECT order_id, match_status FROM v_order_rollup")}
    actual = {
        "missing_po": {o for o, s in rollup.items() if s == "missing_po"},
        "missing_invoice": {o for o, s in rollup.items() if s == "missing_invoice"},
        "fully_matched": {o for o, s in rollup.items() if s == "matched"},
    }
    for key, expected in config.EXPECTED_MATRIX.items():
        assert actual[key] == expected, f"{key}: expected {expected}, got {actual[key]}"
    return {k: sorted(v) for k, v in actual.items()}

if __name__ == "__main__":
    print(run())
    sys.exit(0)
```

- [ ] **Step 5: Run test to verify it passes**

Run: `pytest tests/test_build_validate.py -v`
Expected: PASS (3 tests). If `missing_po`/counts differ, the bug is in the relevant parser (Task 4) — fix the regex there, not the test.

- [ ] **Step 6: Run the pipeline end to end**

Run: `python -m kb.build`
Expected: prints a stats dict; `artifacts/kb.sqlite` exists; no `AssertionError`.

- [ ] **Step 7: Commit**

```bash
git add src/kb/build.py tests/test_build_validate.py tests/conftest.py
git commit -m "feat: build orchestrator with matrix self-check

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 8: MCP server — read layer + SourceRef helpers

**Files:**
- Create: `server.py`, `src/kb/queries.py`
- Test: `tests/test_server_tools.py` (part 1)

**Interfaces:**
- Consumes: read-only `db`.
- Produces (in `src/kb/queries.py`, pure functions taking a `conn`, returning JSON-able dicts — keeps tools thin and testable without MCP):
  - `queries.find_orders(conn, match_status="all", vendor=None, order_id=None, limit=25, offset=0) -> dict`
  - `queries.get_order_documents(conn, order_id, doc_type="all") -> dict`
  - `queries.reconcile_order(conn, order_id, fields=None) -> dict`
  - `queries.search_evidence(conn, query, doc_type="all", order_id=None, limit=5) -> dict`
  - `queries.list_inventory_reports(conn, period=None, category=None, limit=25, offset=0) -> dict`
  - `queries.source_ref(conn, doc_id) -> dict` → `{file, doc_type, doc_id, order_id, page, raw_text}`.

- [ ] **Step 1: Write the failing test** `tests/test_server_tools.py`

```python
from kb import db, queries

def test_find_orders_missing_po(kb_db):
    conn = db.connect(kb_db, readonly=True)
    res = queries.find_orders(conn, match_status="missing_po")
    ids = {o["order_id"] for o in res["orders"]}
    assert ids == {10436, 10687, 10839}
    assert all("sources" in o for o in res["orders"])

def test_get_order_documents_10687(kb_db):
    conn = db.connect(kb_db, readonly=True)
    res = queries.get_order_documents(conn, 10687)
    roles = {d["doc_type"] for d in res["documents"]}
    assert "invoice" in roles and "shipping_order" in roles
    assert "purchase_order" in res["missing_doc_types"]

def test_reconcile_matched_order(kb_db):
    conn = db.connect(kb_db, readonly=True)
    res = queries.reconcile_order(conn, 10248)
    assert res["overall"] in {"matched", "mismatch"}
    assert res["documents_missing"] == []

def test_reconcile_incomplete_order(kb_db):
    conn = db.connect(kb_db, readonly=True)
    res = queries.reconcile_order(conn, 10687)
    assert res["overall"] == "incomplete"
    assert "purchase_order" in res["documents_missing"]

def test_search_evidence_contract(kb_db):
    conn = db.connect(kb_db, readonly=True)
    res = queries.search_evidence(conn, "supply of goods", doc_type="contract")
    assert res["count"] >= 1
    assert res["hits"][0]["source"]["doc_type"] == "contract"

def test_unknown_order_not_found(kb_db):
    conn = db.connect(kb_db, readonly=True)
    res = queries.get_order_documents(conn, 99999)
    assert res["found"] is False
```

- [ ] **Step 2: Run test to verify it fails**

Run: `pytest tests/test_server_tools.py -v`
Expected: FAIL (`ModuleNotFoundError: kb.queries`)

- [ ] **Step 3: Write `src/kb/queries.py`**

```python
ROLE_TABLE = {"invoice": "invoices", "purchase_order": "purchase_orders",
              "shipping_order": "shipping_orders"}

def source_ref(conn, doc_id) -> dict:
    d = conn.execute(
        "SELECT doc_id, doc_type, source_path FROM documents WHERE doc_id=?", (doc_id,)).fetchone()
    if not d:
        return {"doc_id": doc_id, "file": None, "doc_type": None,
                "order_id": None, "page": None, "raw_text": None}
    order_id, raw = None, None
    tbl = ROLE_TABLE.get(d["doc_type"])
    if tbl:
        row = conn.execute(f"SELECT order_id, raw_text FROM {tbl} WHERE doc_id=?", (doc_id,)).fetchone()
        if row:
            order_id = row["order_id"]
            raw = (row["raw_text"] or "")[:160]
    return {"file": d["source_path"], "doc_type": d["doc_type"], "doc_id": d["doc_id"],
            "order_id": order_id, "page": 1, "raw_text": raw}

def find_orders(conn, match_status="all", vendor=None, order_id=None, limit=25, offset=0) -> dict:
    limit = max(1, min(limit, 100))
    where, args = [], []
    if match_status != "all":
        where.append("match_status=?"); args.append(match_status)
    if order_id is not None:
        where.append("order_id=?"); args.append(order_id)
    clause = (" WHERE " + " AND ".join(where)) if where else ""
    total = conn.execute(f"SELECT COUNT(*) c FROM v_order_rollup{clause}", args).fetchone()["c"]
    rows = conn.execute(
        f"SELECT order_id, match_status FROM v_order_rollup{clause} ORDER BY order_id LIMIT ? OFFSET ?",
        (*args, limit, offset)).fetchall()
    orders = []
    for r in rows:
        docs = conn.execute(
            "SELECT role, doc_id FROM order_documents WHERE order_id=?", (r["order_id"],)).fetchall()
        orders.append({
            "order_id": r["order_id"], "match_status": r["match_status"],
            "doc_types_present": sorted({d["role"] for d in docs}),
            "sources": [source_ref(conn, d["doc_id"]) for d in docs],
        })
    return {"count": len(orders), "total_matches": total, "limit": limit, "offset": offset,
            "orders": orders}

def get_order_documents(conn, order_id, doc_type="all") -> dict:
    docs = conn.execute(
        "SELECT role, doc_id FROM order_documents WHERE order_id=?", (order_id,)).fetchall()
    if not docs:
        return {"order_id": order_id, "found": False,
                "message": f"No order {order_id} in the knowledge base.", "documents": []}
    present = {d["role"] for d in docs}
    sel = [d for d in docs if doc_type in ("all", d["role"])]
    return {"order_id": order_id, "found": True,
            "documents": [{"doc_type": d["role"], "doc_id": d["doc_id"],
                           "source": source_ref(conn, d["doc_id"])} for d in sel],
            "missing_doc_types": sorted({"invoice","purchase_order","shipping_order"} - present),
            "sources": [source_ref(conn, d["doc_id"]) for d in sel]}

def _totals(conn, order_id):
    out = {}
    for role, tbl in ROLE_TABLE.items():
        col = "total" if role != "shipping_order" else None
        if col:
            row = conn.execute(f"SELECT total FROM {tbl} WHERE order_id=?", (order_id,)).fetchone()
            out[role] = row["total"] if row else None
        else:
            row = conn.execute(
                "SELECT SUM(line_total) t FROM line_items WHERE order_id=? AND role='shipping_order'",
                (order_id,)).fetchone()
            out[role] = row["t"] if row else None
    return out

def reconcile_order(conn, order_id, fields=None) -> dict:
    base = get_order_documents(conn, order_id)
    if not base.get("found"):
        return {"order_id": order_id, "found": False, "overall": "not_found",
                "documents_present": [], "documents_missing": [], "checks": []}
    present = sorted({d["doc_type"] for d in base["documents"]})
    missing = base["missing_doc_types"]
    totals = _totals(conn, order_id)
    vals = {k: v for k, v in totals.items() if v is not None}
    if "purchase_order" in missing or "invoice" in missing:
        overall = "incomplete"
        status = "unavailable"
    else:
        status = "match" if len(set(round(v, 2) for v in vals.values())) <= 1 else "mismatch"
        overall = "matched" if status == "match" else "mismatch"
    checks = [{"field": "total_amount", "status": status, "values": totals}]
    return {"order_id": order_id, "found": True, "overall": overall,
            "documents_present": present, "documents_missing": missing, "checks": checks,
            "sources": base["sources"]}

def search_evidence(conn, query, doc_type="all", order_id=None, limit=5) -> dict:
    limit = max(1, min(limit, 20))
    sql = ("SELECT f.chunk_id, f.doc_id, snippet(chunks_fts,0,'[',']','…',12) snip, "
           "bm25(chunks_fts) score FROM chunks_fts f "
           "JOIN documents d ON d.doc_id=f.doc_id WHERE chunks_fts MATCH ?")
    args = [query]
    if doc_type != "all":
        sql += " AND d.doc_type=?"; args.append(doc_type)
    sql += " ORDER BY score LIMIT ?"; args.append(limit)
    try:
        rows = conn.execute(sql, args).fetchall()
    except Exception:
        return {"query": query, "count": 0, "hits": [], "message": "No matching evidence found."}
    hits = [{"snippet": r["snip"], "score": r["score"],
             "source": source_ref(conn, r["doc_id"])} for r in rows]
    return {"query": query, "count": len(hits), "hits": hits}

def list_inventory_reports(conn, period=None, category=None, limit=25, offset=0) -> dict:
    limit = max(1, min(limit, 100))
    where, args = [], []
    if period:
        where.append("period=?"); args.append(period)
    if category:
        where.append("category=?"); args.append(category)
    clause = (" WHERE " + " AND ".join(where)) if where else ""
    rows = conn.execute(
        f"SELECT doc_id, period, category, COUNT(*) line_item_count "
        f"FROM inventory_stock{clause} GROUP BY doc_id, period, category "
        f"ORDER BY period LIMIT ? OFFSET ?", (*args, limit, offset)).fetchall()
    reports = [{"report_id": r["doc_id"], "period": r["period"], "category": r["category"],
                "line_item_count": r["line_item_count"],
                "source": source_ref(conn, r["doc_id"])} for r in rows]
    return {"count": len(reports), "reports": reports}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `pytest tests/test_server_tools.py -v`
Expected: PASS (6 tests)

- [ ] **Step 5: Commit**

```bash
git add src/kb/queries.py tests/test_server_tools.py
git commit -m "feat: read-layer query functions with SourceRef citations

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 9: MCP server wiring (FastMCP tools + resources)

**Files:**
- Create: `server.py`
- Test: manual via MCP Inspector (no unit test — `queries` is already covered).

**Interfaces:**
- Consumes: `kb.queries`, `kb.db`, `kb.config`.
- Produces: a runnable stdio MCP server `procurement-kb` exposing the 5 tools + `kb://catalog` + `kb://doc/{doc_id}`.

- [ ] **Step 1: Write `server.py`**

```python
from mcp.server.fastmcp import FastMCP
from kb import db, config, queries

mcp = FastMCP("procurement-kb")

def _conn():
    if not config.DB_PATH.exists():
        raise RuntimeError(f"{config.DB_PATH} not found — run `python -m kb.build` first.")
    return db.connect(config.DB_PATH, readonly=True)

@mcp.tool()
def find_orders(match_status: str = "all", vendor: str | None = None,
                order_id: int | None = None, limit: int = 25, offset: int = 0) -> dict:
    """List/discover orders; filter by match_status (all|matched|missing_po|missing_invoice|partial)."""
    return queries.find_orders(_conn(), match_status, vendor, order_id, limit, offset)

@mcp.tool()
def get_order_documents(order_id: int, doc_type: str = "all") -> dict:
    """Return the documents (invoice/PO/shipping) that support one order_id, plus missing types."""
    return queries.get_order_documents(_conn(), order_id, doc_type)

@mcp.tool()
def reconcile_order(order_id: int, fields: list[str] | None = None) -> dict:
    """Compare invoice/PO/shipping for one order; returns per-field match/mismatch/unavailable verdict."""
    return queries.reconcile_order(_conn(), order_id, fields)

@mcp.tool()
def search_evidence(query: str, doc_type: str = "all",
                    order_id: int | None = None, limit: int = 5) -> dict:
    """Full-text (BM25) search across contract + document text; returns bounded snippets with citations."""
    return queries.search_evidence(_conn(), query, doc_type, order_id, limit)

@mcp.tool()
def list_inventory_reports(period: str | None = None, category: str | None = None,
                           limit: int = 25, offset: int = 0) -> dict:
    """List inventory stock reports and their period/category metadata."""
    return queries.list_inventory_reports(_conn(), period, category, limit, offset)

@mcp.resource("kb://catalog")
def catalog() -> dict:
    conn = _conn()
    return {r["doc_type"]: r["c"] for r in conn.execute(
        "SELECT doc_type, COUNT(*) c FROM documents GROUP BY doc_type")}

@mcp.resource("kb://doc/{doc_id}")
def doc(doc_id: str) -> dict:
    conn = _conn()
    src = queries.source_ref(conn, doc_id)
    chunks = conn.execute("SELECT text FROM chunks WHERE doc_id=? ORDER BY ord", (doc_id,)).fetchall()
    return {"source": src, "text": "\n".join(c["text"] for c in chunks)}

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

- [ ] **Step 2: Build the DB if not present**

Run: `python -m kb.build`
Expected: stats dict, no AssertionError.

- [ ] **Step 3: Launch under MCP Inspector**

Run: `npx @modelcontextprotocol/inspector python server.py`
Expected: Inspector opens; the 5 tools + 2 resources are listed.

- [ ] **Step 4: Manually verify the headline calls in Inspector**

- `find_orders` with `match_status="missing_po"` → orders 10436, 10687, 10839, each with `sources`.
- `reconcile_order` with `order_id=10248` → `overall: matched`/`mismatch`, `documents_missing: []`.
- `reconcile_order` with `order_id=10687` → `overall: incomplete`, missing `purchase_order`.
- `search_evidence` with `query="supply of goods"`, `doc_type="contract"` → ≥1 snippet with a contract `source`.

Capture screenshots into `docs/` — these are the "evidence it works" deliverable.

- [ ] **Step 5: Commit**

```bash
git add server.py
git commit -m "feat: FastMCP stdio server exposing 5 tools + 2 resources

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 10: Documentation deliverables (README, architecture, AI-usage, presentation)

**Files:**
- Modify: `README.md`
- Create: `docs/architecture.md`, `docs/ai-usage.md`, `docs/presentation.md`

**Interfaces:** none (docs only).

- [ ] **Step 1: Update `README.md`** — replace the "Status / planned" sections with real, verified instructions:
  - Prereqs: Python 3.11+, `uv` (or pip), Poppler (`brew install poppler`), Tesseract (`brew install tesseract`), Node (for `npx` Inspector).
  - Setup: `uv sync` (or `pip install -e ".[dev]"`).
  - Run pipeline: `make build` → `artifacts/kb.sqlite`.
  - Run server: `make inspect`.
  - Run tests: `make test`.
  - Short architecture diagram (copy from spec §3) and the tool table (spec §7).

- [ ] **Step 2: Write `docs/architecture.md`** — architecture overview + data-flow overview (offline pipeline stages → store → online MCP read layer), the data model (spec §5), retrieval strategy (spec §6), and the citation strategy (spec §7). Reuse spec prose; this is the standalone reviewer-facing doc.

- [ ] **Step 3: Write `docs/ai-usage.md`** — what was AI-assisted (design via parallel expert agents; code generation; this plan) and how it was validated (TDD per task; the build-time matrix self-check; the Inspector session). One short page.

- [ ] **Step 4: Write `docs/presentation.md`** — a 10–15 min outline covering the brief's required topics: software architecture, data pipeline architecture, MCP server architecture, AI agent/MCP interaction flow, tech-stack decisions, data modeling, ingestion+retrieval strategy, structured-extraction-vs-document-retrieval, cross-document matching, citation strategy, future improvements (embeddings for scale, bbox lineage, more field-level reconciliation, vendor entity resolution for the image invoices).

- [ ] **Step 5: Run the full suite once more**

Run: `make test`
Expected: all tests PASS.

- [ ] **Step 6: Commit and push**

```bash
git add README.md docs/
git commit -m "docs: README, architecture, AI-usage, and presentation outline

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
git push
```

---

## Self-Review

**Spec coverage** (spec §11 deliverables → tasks):
- Pipeline/extraction/OCR → Tasks 3,4,5,7 ✓
- Data model/storage/lineage → Tasks 2,6,8 ✓
- Retrieval (SQL + BM25/FTS5) → Tasks 6,8 ✓
- MCP tools/resources + citations → Tasks 8,9 ✓
- Reproducibility + matrix self-check → Task 7 ✓
- Image-invoice isolation → Tasks 4,7 (`test_image_invoices_unlinked`) ✓
- Evidence with MCP client → Task 9 step 4 ✓
- README / architecture / AI-usage / presentation → Task 10 ✓

**Placeholder scan:** no TBD/TODO; every code step has real code; every test step has real assertions.

**Type consistency:** `SourceRef` fields used in `queries.source_ref` match `models.SourceRef` (Task 1); `queries.*` signatures in Task 8 match the `server.py` tool wrappers in Task 9; `v_order_rollup.match_status` values (`matched|missing_po|missing_invoice|partial`) used consistently in `find_orders`, `validate`, and tests; `reconcile_order.overall` (`matched|mismatch|incomplete|not_found`) is a distinct field from `match_status`, as designed in the spec.

**Known risk to watch during execution:** the regexes in Task 4 are written against the verified `pdftotext -layout` output but may need small tweaks on individual files. The Task 7 matrix self-check is the safety net — if it passes, extraction is correct enough for every rubric question.
