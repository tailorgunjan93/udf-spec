# UDF — Universal Document Format

> **Version 1.0** · [SPEC.md](SPEC.md) · [CHANGELOG.md](CHANGELOG.md) · [License: CC BY 4.0](LICENSE)

UDF is an open, ZIP-based portable document format that stores a document's **text**, **vector embeddings**, and **AI-generated intelligence** (summaries, insights, keywords) in a single `.udf` archive.

---

## Why UDF?

| Problem | UDF Solution |
|---|---|
| PDFs are hard to search semantically | Embeddings baked in at conversion time |
| Re-embedding costs tokens every query | One-time embed; query is free |
| Metadata scattered across systems | Single self-contained archive |
| Vendor lock-in for AI pipelines | Swappable embedding models + LLM providers |
| Large binary blobs mixed with text | Optional `assets/` folder, lazy-loaded |

---

## Quick Start

```bash
pip install docnest-ai
docnest convert report.pdf        # → report.udf
docnest query   report.udf "What is this document about?"
docnest inspect report.udf        # prints structure summary
```

---

## Archive Layout

```
report.udf  (ZIP archive)
├── manifest.json      # format metadata + embedding config
├── catalogue.json     # section index, summaries, embeddings, insights
├── content.json       # full section text, tables, images
└── assets/            # (optional) raw images, attachments
    └── img_0001.png
```

---

## The Three JSON Files

### `manifest.json` — Who am I?

```json
{
  "udf_version": "1.0",
  "doc_id":       "my-report-2024",
  "title":        "Annual Report 2024",
  "source_format":"pdf",
  "created_at":   "2025-01-01T00:00:00+00:00",
  "embedding_model": "huggingface/all-MiniLM-L6-v2",
  "embedding_dims":  384,
  "quantization":    "float16",
  "section_count":   12,
  "intelligence":    true
}
```

### `catalogue.json` — What's inside?

```json
{
  "doc_id":  "my-report-2024",
  "summary": "Annual report summarising financial performance...",
  "insights": ["Revenue grew 18% YoY", "New markets in APAC"],
  "key_numbers": [
    { "label": "Revenue Growth", "value": "18", "unit": "%", "section": "§3" }
  ],
  "section_index": [
    {
      "id": "§1", "title": "Executive Summary", "level": 1,
      "parent_id": null, "children": ["§1.1", "§1.2"],
      "summary": "High-level overview of 2024 performance.",
      "keywords": ["executive", "summary", "performance"],
      "token_count": 312,
      "embedding": "<base64-encoded float16 array>"
    }
  ],
  "embedding_model": "huggingface/all-MiniLM-L6-v2",
  "embedding_dims": 384,
  "quantization": "float16"
}
```

### `content.json` — Full text

```json
{
  "doc_id": "my-report-2024",
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

---

## Quantization Modes

| Mode | Bytes / dim | Precision | Use case |
|---|---|---|---|
| `float32` | 4 | Full | Research / archival |
| `float16` | 2 | High | Default |
| `int8`    | 1 | Good | Storage-constrained |
| `binary`  | 0.125 | Fast approximate | Mobile / edge |

Embeddings are stored as **base64-encoded raw bytes** in `catalogue.json`.

---

## Conformance Levels

| Level | Requirement |
|---|---|
| L1 | Read manifest + catalogue + content; validate `udf_version` |
| L2 | L1 + decode embeddings + perform semantic search |
| L3 | L2 + read/write intelligence fields (summaries, insights) |
| L4 | L3 + write `.udf` archives + support all quantization modes |

See [SPEC.md §14](SPEC.md#14-conformance-levels) for full requirements.

---

## Implementations

| Library | Language | UDF version | Status | Link |
|---|---|---|---|---|
| **docnest-ai** | Python 3.11+ | 1.0 | ✅ Stable | [github.com/tailorgunjan93/DOCNESTd](https://github.com/tailorgunjan93/DOCNESTd) |

See [implementations/README.md](implementations/README.md) for minimal reader/writer examples and how to register your own implementation.

---

## FAQ

**Q: Why ZIP?**  
ZIP is universally supported, streamable, and requires zero extra dependencies on any platform. A `.udf` file can be opened as a ZIP in any file manager or unzipped with standard tools.

**Q: Why not Parquet / HDF5 / SQLite?**  
Those formats require specialised drivers. ZIP + JSON is readable in every language with standard library only.

**Q: Can I store images?**  
Yes. Place them in `assets/` and reference them by filename in `content.json` section `"images"` arrays.

**Q: What if I don't need embeddings?**  
Set `"embedding_model": null`, `"embedding_dims": 0`, and all `"embedding"` fields to `null`. The file is still valid UDF.

**Q: Is UDF stable?**  
Version 1.0 is stable. See [CHANGELOG.md](CHANGELOG.md) and [SPEC.md §15](SPEC.md#15-versioning-and-forward-compatibility) for the version policy.

---

## License

[Creative Commons Attribution 4.0 International](LICENSE) — use freely, attribute the spec.
