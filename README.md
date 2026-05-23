# UDF — Universal Document Format

> **Version 1.1** · [SPEC.md](SPEC.md) · [CHANGELOG.md](CHANGELOG.md) · [License: CC BY 4.0](LICENSE)

UDF is an open, ZIP-based portable document format that stores a document's **text**, **vector embeddings**, and **AI-generated intelligence** (summaries, insights, keywords) in a single `.udf` archive. Built for AI pipelines and RAG systems — not for end-user document management.

> **Who is UDF for?**  
> UDF is an **AI-first, developer-first format.** It is designed for ML engineers, RAG pipeline builders, and researchers who need a portable, queryable knowledge base — not for business users who open documents directly. Human-readable output is produced via `docnest view` (HTML) or the application layer built on top.  
> See [§ Intended Audience](#intended-audience) for the full breakdown.

---

## Why UDF?

| Problem | UDF Solution |
|---|---|
| PDFs are hard to search semantically | Embeddings baked in at conversion time |
| Re-embedding costs tokens every query | One-time embed; query is free |
| Metadata scattered across systems | Single self-contained archive |
| Vendor lock-in for AI pipelines | Swappable embedding models + LLM providers |
| Large binary blobs slow to load | `embeddings.bin` loaded lazily only when searched |
| No organisational context on documents | `owner`, `department`, `tags`, `access_roles` fields |
| Multi-document discovery is slow | Library layer with cross-document keyword index |

---

## Quick Start

```bash
pip install docnest-ai

# Convert a document
docnest convert report.pdf --owner "Alice" --department "Finance" --tags "quarterly,2024"

# Semantic search
docnest query report.udf "What is this document about?"

# Inspect structure
docnest inspect report.udf

# View as HTML (opens in browser)
docnest view report.udf

# Multi-document library
docnest library init ./docs/
docnest library add  ./docs/ report.udf
docnest library search ./docs/ "quarterly revenue"
```

---

## Archive Layout

```
report.udf  (ZIP archive)
├── manifest.json      # format metadata, embedding config, organisational metadata
├── catalogue.json     # section index, summaries, keywords, insights
├── content.json       # full section text, tables, image refs
├── embeddings.bin     # (optional) compact binary embedding blob — loaded lazily
└── assets/            # (optional) raw images, attachments
    └── img_0001.png
```

When `embeddings.bin` is present, embeddings are stored as a flat binary blob in
`section_index` order — ~87% smaller than base64 strings in `catalogue.json`.

---

## The Three JSON Files

### `manifest.json` — Who am I?

```json
{
  "udf_version":      "1.0",
  "doc_id":           "annual-report-2024",
  "title":            "Annual Report 2024",
  "source_format":    "pdf",
  "created_at":       "2026-01-01T00:00:00+00:00",
  "embedding_model":  "huggingface/all-MiniLM-L6-v2",
  "embedding_dims":   384,
  "quantization":     "float16",
  "section_count":    12,
  "intelligence":     true,
  "embedding_format": "binary",
  "owner":            "Alice Johnson",
  "department":       "Finance",
  "tags":             ["quarterly", "2024", "revenue"],
  "access_roles":     ["finance", "executives"],
  "version":          "1.0",
  "producer":         "docnest-ai 1.0"
}
```

### `catalogue.json` — What's inside?

```json
{
  "doc_id":  "annual-report-2024",
  "summary": "Annual report summarising financial performance for 2024...",
  "insights": ["Revenue grew 18% YoY", "New markets opened in APAC"],
  "key_numbers": [
    { "label": "Revenue Growth", "value": "18", "unit": "%", "section": "§3" }
  ],
  "section_index": [
    {
      "id": "§1", "title": "Executive Summary", "level": 1,
      "parent_id": null, "children": ["§1.1", "§1.2"],
      "summary": "High-level overview of 2024 performance.",
      "keywords": ["executive", "summary", "performance", "revenue"],
      "token_count": 312
    }
  ],
  "embedding_model": "huggingface/all-MiniLM-L6-v2",
  "embedding_dims":  384,
  "quantization":    "float16"
}
```

> **Note:** When `embedding_format` is `"binary"`, per-section `"embedding"` fields are omitted from `catalogue.json`. All embeddings live in `embeddings.bin` in `section_index` order.

### `content.json` — Full text

```json
{
  "doc_id": "annual-report-2024",
  "sections": {
    "§1": {
      "title": "Executive Summary",
      "level": 1,
      "text":  "This report covers...",
      "tables": [],
      "images": []
    }
  }
}
```

---

## Section ID System

Sections are identified by a `§` prefix and dot-notation depth:

```
§1          root / document title
  §1.1      first child
  §1.2      second child
    §1.2.1  grandchild
§2          second root section
```

Both humans and AI navigate the same `§id` index — the catalogue is the shared map.

---

## Quantization Modes

| Mode | Bytes/dim | 384-dim stride | Precision | Use case |
|---|---|---|---|---|
| `float32` | 4 | 1 536 B | Full | Research / archival |
| `float16` | 2 | 768 B | High | **Default** |
| `int8` | 1 | 384 B | Good | Storage-constrained |
| `binary` | 0.125 | 48 B | Approximate | Mobile / edge |

Embeddings are stored as a **compact binary blob** (`embeddings.bin`) when `embedding_format = "binary"`, or as **Base64-encoded raw bytes** in `catalogue.json` for backward compatibility.

---

## Embedding Storage: Binary vs Base64

| | Binary (`embeddings.bin`) | Base64 (catalogue) |
|---|---|---|
| **catalogue.json size** | ~87% smaller | Baseline |
| **Load model** | Lazy — only on search | Decoded at open time |
| **RAM at open** | Near-zero | Full matrix |
| **numpy read** | `np.frombuffer()` zero-copy | Base64 decode + cast |
| **Backward compat** | v1.0 readers ignore the file | All versions |

---

## Conformance Levels

| Level | Requirement |
|---|---|
| L1 | Read manifest + catalogue + content; validate `udf_version` |
| L2 | L1 + decode embeddings (both binary and base64) + semantic search |
| L3 | L2 + read intelligence fields + multi-layer query routing |
| L4 | L3 + write `.udf` archives + all quantization modes + smart compression |

See [SPEC.md §14](SPEC.md#14-conformance-levels) for full requirements.

---

## Intended Audience

UDF is an **AI-first, developer-first format.** It is not a consumer document format like PDF or DOCX.

| User | Fits UDF? | Why |
|---|---|---|
| ML / RAG engineer | ✅ Primary | Builds pipelines on top of `.udf` — this is the core use case |
| Researcher | ✅ Yes | Converts paper collections to queryable, portable knowledge bases |
| Backend developer | ✅ Yes | UDF as the storage layer behind a chatbot or search product |
| Data scientist | ✅ Yes | Extracts structured tables, summaries, key numbers via Python API |
| Business analyst | ⚠️ Via app | Should interact with a product UI or `docnest view` HTML output — not `.udf` directly |
| Document manager | ⚠️ Via app | Same — UDF is the engine, not the interface |
| End consumer | ❌ Not directly | Not designed for this; the application layer bridges the gap |

**Bridges for non-developer users:**
- `docnest view report.udf` — generates a fully human-readable, self-contained HTML page (opens in browser)
- `docnest inspect report.udf` — prints section tree, metadata, key numbers in plain text
- Application UIs built on top of DOCNEST (chatbots, search portals, dashboards)
- VS Code extension: [udf-reader-vscode](https://github.com/tailorgunjan93/udf-reader-vscode) — browse `.udf` files directly in the editor

The `.udf` format is intentionally optimised for AI query performance, not human direct-opening. Changing this would compromise the embedding storage model and section index structure that make DocNest fast and accurate.

---

## Accuracy (v7 eval — May 2026)

The reference implementation `docnest-ai` achieves **9.55 / 10** honest accuracy across **88 questions, 10 documents, 5 formats**:

| Format | Score | Pass Rate |
|---|---|---|
| DOCX · HTML · MD | 9.9–10.0 / 10 | 100% |
| XLSX | 8.8 / 10 | 87% |
| PDF (scientific papers) | 7.8–10.0 / 10 | 60–100% |
| **Overall** | **9.55 / 10** | **95.5%** |

4 real retrieval errors in 88 questions. Zero LLM hallucinations. Evaluator: Cerebras `qwen-3-235b-a22b-instruct-2507`.

---

## Implementations

| Library | Language | UDF version | Conformance | Status | Link |
|---|---|---|---|---|---|
| **docnest-ai** | Python 3.11+ | 1.1 | L4 | ✅ Stable | [github.com/tailorgunjan93/docnest](https://github.com/tailorgunjan93/docnest) |

See [implementations/README.md](implementations/README.md) for minimal reader/writer examples and how to register your own implementation.

---

## FAQ

**Q: Why ZIP?**  
ZIP is universally supported, streamable, and requires zero extra dependencies on any platform. A `.udf` file can be opened as a ZIP in any file manager or unzipped with standard tools.

**Q: Why not Parquet / HDF5 / SQLite?**  
Those formats require specialised drivers. ZIP + JSON is readable in every language with standard library only.

**Q: Can I store images?**  
Yes. Place them in `assets/` and reference them by filename in `content.json` section `"images"` arrays. PNG/JPEG/WEBP are stored uncompressed inside the ZIP (already compressed formats).

**Q: What if I don't need embeddings?**  
Set `"embedding_model": null`, `"embedding_dims": 0`, and omit `embeddings.bin`. The file is still valid UDF.

**Q: What is embeddings.bin?**  
A flat binary blob of all section embeddings concatenated in `section_index` order, each padded to the same stride. Loaded lazily on first search — not at archive-open time.

**Q: What are DocMeta fields?**  
`owner`, `department`, `tags`, `access_roles`, `version`, and `last_updated` — optional organisational fields in `manifest.json` (added in v1.1). They let enterprises tag, filter, and access-control documents in a library without opening each archive.

**Q: Is UDF stable?**  
Version 1.0 is stable. Version 1.1 adds only backward-compatible optional fields. See [CHANGELOG.md](CHANGELOG.md) and [SPEC.md §15](SPEC.md#15-versioning-and-forward-compatibility) for the version policy.

---

## License

[Creative Commons Attribution 4.0 International](LICENSE) — use freely, attribute the spec.
