# Caseware Procurement & Inventory MCP Knowledge Base

A local data pipeline + **MCP server** over a procurement/inventory knowledge base
(invoices, purchase orders, shipping orders, inventory reports, contracts) that lets an
MCP-compatible AI agent retrieve, compare, and reason across documents and return
**grounded answers with source citations**.

Built for the Caseware "Senior AI Data Software Developer" take-home.

## Status

🚧 Design complete; implementation in progress.

- **Design spec:** [`docs/superpowers/specs/2026-06-18-caseware-mcp-design.md`](docs/superpowers/specs/2026-06-18-caseware-mcp-design.md)
- **Implementation plan:** _coming next_ (`docs/superpowers/plans/`)

## Approach (summary)

Two layers with hard separation:

- **Offline pipeline** (`build.py`) — extracts text/OCR from PDFs and image invoices,
  parses structured fields, chunks the contract, and loads everything into a single
  `kb.sqlite` (SQLite + FTS5). Reproducible, wipe-and-rebuild, with a build-time self-check.
- **Online MCP server** (`server.py`) — a thin, read-only `FastMCP` (stdio) server exposing
  ~5 order-centric tools. It only runs parameterized SQL/FTS queries against the pre-built
  store; **no extraction at request time**.

Retrieval is **hybrid**: SQL/metadata joins on `order_id` for relational questions
("which invoices are missing a PO?", "mismatches between invoice/PO/shipping?") and
**BM25 / SQLite FTS5** text search for the contract and free-text evidence.

## Quickstart (planned)

```bash
make build     # run the pipeline -> artifacts/kb.sqlite
make inspect   # launch the MCP server under the MCP Inspector
```

See the design spec for the full architecture, data model, retrieval strategy, MCP tool
descriptions, and citation strategy.
