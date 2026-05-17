# Changelog

All notable changes to the UDF specification are documented here.

This project adheres to [Semantic Versioning](https://semver.org/) (MAJOR.MINOR).

---

## [1.1] — 2026-05-17

### Added — Embedding Storage

- **`embeddings.bin`** — optional binary embedding blob at archive root. Stores all section embeddings contiguously in `section_index` order, one `stride`-byte record per section. Sections without an embedding use `stride` zero bytes.
- **`manifest.embedding_format`** — new optional field (`"base64"` | `"binary"`, default `"base64"`). When `"binary"`, the reader loads embeddings from `embeddings.bin` instead of `catalogue.json` per-section `"embedding"` fields.
- Lazy loading guidance: readers SHOULD NOT decode embeddings at archive-open time; load `embeddings.bin` only when a search is performed.
- Smart ZIP compression rules: DEFLATE level 9 for JSON/text, DEFLATE level 1 for `embeddings.bin`, ZIP_STORED for pre-compressed images (PNG, JPEG, WEBP, GIF).
- Measured benefits over base64: ~87% smaller `catalogue.json`, ~63% less peak RAM on archive open.

### Added — Organisational Metadata (DocMeta)

New optional fields in `manifest.json`:

| Field | Type | Description |
|---|---|---|
| `owner` | string | Person or system that owns this document |
| `department` | string | Organisational unit (e.g. `"Engineering"`, `"Finance"`) |
| `tags` | array of strings | Free-form labels for search and filtering |
| `access_roles` | array of strings | Roles permitted to read; `["*"]` = unrestricted |
| `version` | string | Document version string (e.g. `"1.0"`, `"2024-Q3"`) |
| `last_updated` | string | ISO 8601 date or datetime of last content update |

### Added — Library Layer (§17)

- New `library.json` format for indexing multiple `.udf` archives in a directory.
- `keywords_bag` field: union of all section keywords across a document, enabling fast cross-document keyword search without opening each archive.
- Two-phase search: filter by `keywords_bag` (no I/O), then open only matching archives for semantic ranking.
- `docnest library init/add/list/search/remove` CLI commands.

### Added — HTML Viewer

- `docnest view <file.udf>` command generates a self-contained HTML page with:
  - Sidebar table of contents with scroll-sync (Intersection Observer)
  - Hierarchical section display with heading levels
  - Table rendering (`<table>` with `<thead>` / `<tbody>`)
  - Keyword badges per section
  - Metadata bar: owner, department, embedding model, quantization, created date
- No external dependencies — single HTML file, no CDN required.

### Added — CLI

- `docnest view <udf> [--output <html>] [--no-open]`
- `docnest library init <dir>`
- `docnest library add  <dir> <udf>`
- `docnest library list <dir>`
- `docnest library search <dir> <query>`
- `docnest library remove <dir> <doc_id>`
- `docnest convert` gained: `--owner`, `--department`, `--tags`, `--access-roles`, `--doc-version`

### Fixed — doc_id and title derivation

- Filename-to-slug and filename-to-title functions now split CamelCase correctly.
  - `GunjanTailor.pdf` → `doc_id: "gunjan-tailor"`, `title: "Gunjan Tailor"` (was `"gunjantailor"`)
  - `SampleReport2024` → `doc_id: "sample-report-2024"`, `title: "Sample Report 2024"`

### Updated — SPEC.md

- §3: Added compression level guidance
- §4: Added `embeddings.bin` to file layout
- §5: Added DocMeta fields; added `embedding_format`; added `doc_id` derivation rule
- §6: Clarified that `embedding` per-section is omitted when `embedding_format = "binary"`
- §9: Replaced single encoding section with §9.1 (Base64/legacy) and §9.2 (Binary blob/preferred)
- §10: Added stride formula and summary table
- §12: Added image compression guidance
- §13: Added `embeddings.bin` wrong-size error handling
- §14: L2 now requires binary format support and lazy loading; L3 now includes DocMeta; L4 now includes smart compression
- §15: Added v1.0→v1.1 migration note
- §17: New — Library Layer

---

## [1.0] — 2026-05-17

Initial public release of the Universal Document Format specification.

### Defined

- ZIP-based container format with `.udf` extension
- Three required JSON files: `manifest.json`, `catalogue.json`, `content.json`
- Optional `assets/` folder for binary attachments
- `§id` hierarchical section identifier system (dot-notation, `§` prefix)
- Four quantization modes: `float32`, `float16`, `int8`, `binary`
- Base64 raw-bytes embedding encoding in `catalogue.json`
- Intelligence fields: `summary`, `insights`, `key_numbers`, `keywords`
- Four conformance levels: L1 (Basic Reader) through L4 (Full Writer)
- Forward-compatibility rules for unknown fields
- Extension rules via `"custom"` key

### Reference Implementation

- [docnest-ai](https://github.com/tailorgunjan93/DOCNESTd) — Python 3.11+, Level 4 conformant
