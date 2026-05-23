# UDF Specification — Version 1.1

**Status:** Stable  
**Published:** 2026-05-17  
**Updated:** 2026-05-23 (v1.1)  
**Author:** Gunjan Tailor  
**License:** [CC BY 4.0](LICENSE)

---

## Table of Contents

1. [Terminology](#1-terminology)
2. [Design Goals](#2-design-goals)
3. [Container Format](#3-container-format)
4. [File Layout](#4-file-layout)
5. [manifest.json Schema](#5-manifestjson-schema)
6. [catalogue.json Schema](#6-cataloguejson-schema)
7. [content.json Schema](#7-contentjson-schema)
8. [Section ID System](#8-section-id-system)
9. [Embedding Encoding](#9-embedding-encoding)
10. [Quantization Modes](#10-quantization-modes)
11. [Intelligence Fields](#11-intelligence-fields)
12. [Assets Folder](#12-assets-folder)
13. [Error Handling](#13-error-handling)
14. [Conformance Levels](#14-conformance-levels)
15. [Versioning and Forward Compatibility](#15-versioning-and-forward-compatibility)
16. [Extension Rules](#16-extension-rules)
17. [Library Layer](#17-library-layer)

---

## 1. Terminology

| Term | Meaning |
|---|---|
| **UDF archive** | A ZIP file with the `.udf` extension |
| **Section** | A named, contiguous portion of a document (heading + body) |
| **§id** | A string identifier for a section, e.g. `§1.2` |
| **Embedding** | A fixed-length float vector representing semantic content |
| **Quantization** | A lossy compression of float vectors to smaller dtypes |
| **Intelligence** | AI-generated metadata: summaries, insights, key numbers, keywords |
| **DocMeta** | Organizational metadata: owner, department, tags, access control |
| **Reader** | Software that opens and queries `.udf` files |
| **Writer** | Software that produces `.udf` files |
| **Library** | A `library.json` index that tracks multiple `.udf` archives |

The key words MUST, MUST NOT, REQUIRED, SHOULD, SHOULD NOT, MAY, and OPTIONAL in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## 2. Design Goals

### Primary goals (AI pipeline)
1. **Zero-dependency readability** — The three core JSON files MUST be parseable with standard library only (Python `json`, Node.js `JSON.parse`, etc.).
2. **Single-file portability** — One `.udf` file contains everything needed to query a document offline.
3. **Swappable AI** — The format records which model produced the embeddings but does not mandate a specific provider.
4. **Forward-compatible** — New fields MUST be ignored by readers that do not recognise them.
5. **Memory-efficient** — Embeddings SHOULD be stored as a compact binary blob (`embeddings.bin`) loaded lazily on demand, not decoded at archive-open time.
6. **Organisational** — Documents SHOULD carry ownership, department, and access-control metadata so they can be managed at enterprise scale.
7. **Query performance** — Structure and pre-computed intelligence MUST be optimised for AI retrieval accuracy, not for human direct-reading.

### Secondary goal (developer inspection)
8. **Developer-inspectable** — A developer SHOULD be able to `unzip -p report.udf manifest.json | python -m json.tool` and read the output. This is a debugging and auditing aid, not the primary consumption path.

### Non-goal: direct human consumption
UDF is **not** designed to be opened directly by end users (business analysts, document managers, or consumers). Human-readable output is delegated to the **application layer**:
- `docnest view report.udf` → self-contained HTML (opens in browser)
- `docnest inspect report.udf` → plain-text section tree
- VS Code extension, desktop apps, or product UIs built on top of a UDF reader

Attempting to make `.udf` a direct end-user format would require compromising the embedding storage model (flat binary blob) and section index structure that make retrieval fast and accurate. This trade-off is intentional.

---

## 3. Container Format

A `.udf` file is a **ZIP archive** (PKZIP format, version 2.0 or higher).

- Compression method: DEFLATE (method 8) is RECOMMENDED. Store (method 0) is permitted.
- Compression levels: Writers SHOULD use **DEFLATE level 9** for JSON/text entries and **DEFLATE level 1** for binary blobs (`embeddings.bin`). Already-compressed image formats (PNG, JPEG, WEBP, GIF) SHOULD use Store (method 0).
- Encoding: All JSON entry names and content MUST be UTF-8 encoded.
- JSON formatting: Compact (no indentation, minimal separators) is RECOMMENDED for size; pretty-printed is also valid.
- Max archive size: No formal limit. Implementations SHOULD handle archives up to 4 GB (ZIP64 for larger).
- The file extension MUST be `.udf` (lowercase).
- The MIME type is `application/x-udf` (unofficial; register with your server as needed).

---

## 4. File Layout

```
<name>.udf
├── manifest.json       REQUIRED
├── catalogue.json      REQUIRED
├── content.json        REQUIRED
├── embeddings.bin      OPTIONAL — binary embedding blob (preferred over base64 in catalogue)
└── assets/             OPTIONAL
    ├── img_0001.png
    ├── img_0002.jpg
    └── attachment.xlsx
```

### Rules

- `manifest.json`, `catalogue.json`, and `content.json` MUST appear at the root level of the archive (no subdirectory prefix).
- `embeddings.bin` is OPTIONAL. When present it supersedes `embedding` fields in `catalogue.json → section_index`. See §9.
- The `assets/` folder is OPTIONAL. Files inside it MAY use any naming convention.
- A reader that encounters unknown files at the root level MUST ignore them without error.
- Future versions of this spec MAY define additional root-level files.

---

## 5. manifest.json Schema

`manifest.json` describes the archive itself, embedding configuration, and organisational metadata.

### Full Schema

```json
{
  "udf_version":      "<string>",        // REQUIRED — e.g. "1.0"
  "doc_id":           "<string>",        // REQUIRED — unique slug identifier
  "title":            "<string>",        // REQUIRED — human-readable title
  "source_format":    "<string>",        // REQUIRED — "pdf", "docx", "txt", "html", "md", …
  "created_at":       "<ISO-8601>",      // REQUIRED — UTC timestamp
  "embedding_model":  "<string|null>",   // REQUIRED — e.g. "huggingface/all-MiniLM-L6-v2" or null
  "embedding_dims":   <integer|null>,    // REQUIRED — number of dimensions, or null if no embeddings
  "quantization":     "<string|null>",   // REQUIRED — "float32"|"float16"|"int8"|"binary"|null
  "section_count":    <integer>,         // REQUIRED — total number of sections
  "intelligence":     <boolean>,         // REQUIRED — true if AI intelligence fields are populated

  "embedding_format": "<string>",        // OPTIONAL (v1.1) — "base64" | "binary" (default: "base64")
  "language":         "<string>",        // OPTIONAL — BCP-47 language tag, e.g. "en", "fr"
  "source_path":      "<string>",        // OPTIONAL — original file path at conversion time
  "producer":         "<string>",        // OPTIONAL — name + version of the software that wrote this file

  "owner":            "<string>",        // OPTIONAL (v1.1) — person or system that owns this document
  "department":       "<string>",        // OPTIONAL (v1.1) — organisational unit
  "tags":             ["<string>"],      // OPTIONAL (v1.1) — free-form labels for search and filtering
  "access_roles":     ["<string>"],      // OPTIONAL (v1.1) — roles permitted to read this archive; ["*"] = public
  "version":          "<string>",        // OPTIONAL (v1.1) — document version string, e.g. "1.0", "2024-Q3"
  "last_updated":     "<ISO-8601|string>", // OPTIONAL (v1.1) — last content update date

  "custom":           <object>           // OPTIONAL — arbitrary extension data
}
```

### Field Constraints

| Field | Constraint |
|---|---|
| `udf_version` | MUST match `^\d+\.\d+$`. Readers MUST reject archives where the major version is higher than supported. |
| `doc_id` | MUST be non-empty. SHOULD be URL-safe (alphanumeric, `-`, `_`). SHOULD be derived from filename by splitting CamelCase and replacing spaces/underscores with hyphens. MUST match `doc_id` in `catalogue.json` and `content.json`. |
| `created_at` | MUST be a valid ISO 8601 datetime with timezone. |
| `quantization` | MUST be one of `"float32"`, `"float16"`, `"int8"`, `"binary"`, or `null`. |
| `embedding_dims` | MUST be a positive integer when `embedding_model` is non-null. |
| `section_count` | MUST equal the number of entries in `catalogue.json → section_index`. |
| `embedding_format` | When `"binary"`, a valid `embeddings.bin` entry MUST be present in the archive. When absent or `"base64"`, embeddings are stored in `catalogue.json`. |
| `tags` | MUST be an array of lowercase strings when present. |
| `access_roles` | The special value `"*"` means unrestricted access. |

---

## 6. catalogue.json Schema

`catalogue.json` is the document's index. It contains per-section metadata and (optionally) per-section embeddings, plus document-level intelligence.

### Full Schema

```json
{
  "doc_id":    "<string>",      // REQUIRED — must match manifest.doc_id
  "title":     "<string>",      // REQUIRED
  "source":    "<string>",      // OPTIONAL — file path or URL
  "language":  "<string>",      // OPTIONAL — BCP-47
  "summary":   "<string>",      // OPTIONAL — document-level summary (AI or manual)
  "insights":  ["<string>"],    // OPTIONAL — bulleted insight strings
  "key_numbers": [              // OPTIONAL
    {
      "label":   "<string>",    // REQUIRED within object
      "value":   "<string>",    // REQUIRED — stored as string to preserve precision
      "unit":    "<string>",    // OPTIONAL
      "section": "<§id>"        // OPTIONAL — source section
    }
  ],
  "section_index": [            // REQUIRED — array of section descriptors
    {
      "id":         "<§id>",      // REQUIRED
      "title":      "<string>",   // REQUIRED
      "level":      <integer>,    // REQUIRED — 1 = top-level, 2 = sub, etc.
      "parent_id":  "<§id>|null", // REQUIRED — null for root sections
      "children":   ["<§id>"],    // REQUIRED — may be empty array
      "summary":    "<string>",   // OPTIONAL — section-level summary
      "keywords":   ["<string>"], // OPTIONAL — lower-cased keyword list
      "token_count":<integer>,    // OPTIONAL — approx token count of section text
      "embedding":  "<base64>|null" // OPTIONAL — only when embedding_format is "base64"
    }
  ],
  "embedding_model": "<string|null>",  // REQUIRED — must match manifest
  "embedding_dims":  <integer|null>,   // REQUIRED — must match manifest
  "quantization":    "<string|null>",  // REQUIRED — must match manifest
  "custom":          <object>          // OPTIONAL
}
```

### section_index Rules

- `section_index` MUST be present and MUST be a non-empty array (at least one section).
- The order of entries in `section_index` MUST reflect document reading order. The `embeddings.bin` blob (§9) is ordered identically.
- `id` values MUST be unique within the archive.
- `parent_id` MUST reference an `id` that appears elsewhere in `section_index`, or be `null`.
- `children` MUST list exactly the `id` values whose `parent_id` points to this section. An empty array `[]` is valid.
- When `embedding_format` is `"base64"` (or absent): if `embedding` is non-null, its decoded byte length MUST equal `embedding_dims × bytes_per_element` (see §10).
- When `embedding_format` is `"binary"`: the `embedding` field SHOULD be omitted or set to `null`; embeddings are read from `embeddings.bin`.

---

## 7. content.json Schema

`content.json` stores the full text and structured content (tables, image refs) for every section.

### Full Schema

```json
{
  "doc_id": "<string>",     // REQUIRED — must match manifest.doc_id
  "sections": {             // REQUIRED — object keyed by §id
    "<§id>": {
      "title":  "<string>", // REQUIRED
      "level":  <integer>,  // REQUIRED
      "text":   "<string>", // REQUIRED — full plain text of section body; may be ""
      "tables": [           // OPTIONAL — list of extracted tables
        {
          "caption": "<string>",      // OPTIONAL
          "headers": ["<string>"],    // OPTIONAL
          "rows":    [["<string>"]]   // REQUIRED within table object
        }
      ],
      "images": [           // OPTIONAL — list of image references
        {
          "asset":   "<filename>",  // REQUIRED — filename within assets/ folder
          "caption": "<string>",    // OPTIONAL
          "alt":     "<string>"     // OPTIONAL — alt text / OCR output
        }
      ],
      "custom": <object>    // OPTIONAL
    }
  }
}
```

### sections Rules

- Every `§id` present in `catalogue.json → section_index` MUST have a corresponding key in `content.json → sections`.
- `text` MUST be present (use `""` for empty sections such as title-only nodes).
- Table `rows` is an array of arrays; each inner array is a row of cell strings.
- Image `asset` filenames MUST correspond to files present in the `assets/` folder.

---

## 8. Section ID System

Section identifiers follow a hierarchical dot-notation scheme prefixed with `§` (U+00A7, section sign).

### Grammar

```
section_id  ::= "§" level ("." level)*
level       ::= [1-9][0-9]*
```

### Examples

```
§1           Root section 1
§1.1         First child of §1
§1.2         Second child of §1
§1.2.1       First grandchild
§2           Root section 2
§10          Root section 10 (multi-digit OK)
```

### Rules

- IDs MUST NOT start with `§0` (zero is reserved).
- IDs MUST NOT contain leading zeros within any level component (e.g. `§1.01` is invalid; use `§1.1`).
- IDs MUST be unique within an archive.
- The depth of nesting is unlimited, but implementations SHOULD handle at least 6 levels.
- Writers SHOULD assign IDs in document reading order starting from `§1`.

---

## 9. Embedding Encoding

UDF supports two embedding storage formats, indicated by `manifest.embedding_format`:

| `embedding_format` | Storage location | Description |
|---|---|---|
| `"base64"` (or absent) | `catalogue.json → section_index[*].embedding` | Per-section Base64 string; legacy format |
| `"binary"` | `embeddings.bin` root entry | Contiguous binary blob; preferred for efficiency |

### 9.1 Base64 Format (Legacy)

When `embedding_format` is `"base64"` or absent:

1. Obtain the float vector (list of `embedding_dims` floats).
2. Quantize according to the `quantization` mode (see §10).
3. Encode the resulting byte array using Base64 (standard alphabet, no line wrapping, padding optional).
4. Store the Base64 string in `catalogue.json → section_index[i].embedding`.

Decoding is the reverse: decode Base64 → interpret bytes per quantization mode.

### 9.2 Binary Blob Format (Preferred)

When `embedding_format` is `"binary"`, all section embeddings are stored as a single contiguous binary file `embeddings.bin` at the archive root.

**Structure:**

```
embeddings.bin = embedding[0] || embedding[1] || … || embedding[N-1]
```

Where:
- `N` = `manifest.section_count`
- `embedding[i]` = quantized bytes for section `section_index[i]`, in order
- Each embedding occupies exactly `stride` bytes, where `stride` is defined by the quantization mode (see §10)
- Sections without a valid embedding MUST still occupy `stride` bytes of zeros (`\x00 * stride`)

**Total file size:** `N × stride` bytes exactly. Readers MUST reject `embeddings.bin` entries of wrong size.

**Reader algorithm:**

```python
raw  = zf.read("embeddings.bin")
n    = manifest["section_count"]
dims = manifest["embedding_dims"]
# stride depends on quantization mode (see §10)
mat  = np.frombuffer(raw, dtype=numpy_dtype).reshape(n, -1).astype(np.float32)
# mat[i] is the embedding for section_index[i]
```

**Advantages over base64:**
- ~87% smaller catalogue.json (no repeated base64 strings)
- Zero-copy `np.frombuffer()` reshape
- Binary can be memory-mapped for very large archives
- Lazy loading: load `embeddings.bin` only when a search is performed

---

## 10. Quantization Modes

### float32

- NumPy dtype: `float32` (IEEE 754 single precision)
- Bytes per dimension: 4
- Stride: `embedding_dims × 4`
- Endianness: little-endian

### float16

- NumPy dtype: `float16` (IEEE 754 half precision)
- Bytes per dimension: 2
- Stride: `embedding_dims × 2`
- Endianness: little-endian
- **Default mode** for docnest-ai 1.0+

### int8

- NumPy dtype: `int8`
- Bytes per dimension: 1
- Stride: `embedding_dims × 1`
- Encoding: scale each float `v` to `round(v × 127)`, clamped to `[-127, 127]`
- Decoding: divide each int8 value by `127.0`

### binary

- NumPy dtype: `uint8` bit-packed
- Bytes per dimension: 1/8
- Stride: `ceil(embedding_dims / 8)` bytes
- Encoding: threshold each float at 0; bit `i` = 1 if `v[i] > 0`, else 0. Pack bits MSB-first.
- Decoding: unpack bits; interpret 1 as `+1.0`, 0 as `-1.0` (or `0.0` — implementation choice).

### Summary Table

| Mode | Bytes/dim | Stride (384-dim) | Precision | Use case |
|---|---|---|---|---|
| `float32` | 4 | 1536 B | Full | Research / archival |
| `float16` | 2 | 768 B | High | Default |
| `int8` | 1 | 384 B | Good | Storage-constrained |
| `binary` | 0.125 | 48 B | Approximate | Mobile / edge |

---

## 11. Intelligence Fields

Intelligence fields are AI-generated annotations. They are OPTIONAL but MUST follow the schema when present.

| Field | Location | Type | Description |
|---|---|---|---|
| `summary` (doc) | `catalogue.json` | string | Document-level abstractive summary |
| `summary` (section) | `section_index[*]` | string | Section-level abstractive summary |
| `insights` | `catalogue.json` | array of strings | Key bullet-point findings |
| `key_numbers` | `catalogue.json` | array of objects | Numeric facts with labels and units |
| `keywords` | `section_index[*]` | array of strings | Lower-cased topical keywords |

When `manifest.intelligence` is `true`, implementations SHOULD populate at least `summary` and `keywords` for all sections.

When `manifest.intelligence` is `false`, all intelligence fields SHOULD be `""` / `[]` / `null`.

---

## 12. Assets Folder

The `assets/` folder holds binary files referenced from `content.json → sections[*].images[*].asset`.

- All asset filenames MUST be unique within the `assets/` folder.
- Supported formats: PNG, JPEG, GIF, WEBP (images), XLSX, CSV (attachments). Other formats are permitted.
- Image formats (PNG, JPEG, WEBP, GIF) are already compressed and SHOULD be stored with `ZIP_STORED` (method 0) to avoid wasting CPU.
- Readers MAY skip loading assets if they are not needed for the query.
- Writers SHOULD use sequential naming: `img_0001.png`, `img_0002.jpg`, etc.

---

## 13. Error Handling

### Missing Required Files

A reader MUST raise an error if any of `manifest.json`, `catalogue.json`, or `content.json` is absent.

### Version Mismatch

A reader that supports UDF major version `N` MUST reject archives with major version > `N` with a clear error message.

### Inconsistent doc_id

A reader MUST raise an error if `doc_id` in `manifest.json`, `catalogue.json`, and `content.json` do not all match.

### Corrupt or Wrong-Size embeddings.bin

A reader that finds `embeddings.bin` with a byte count that does not equal `section_count × stride` MUST raise a warning and SHOULD fall back to base64 embeddings in `catalogue.json` if present, or treat all embeddings as null.

### Corrupt Base64 Embeddings

A reader that decodes an embedding of unexpected byte length SHOULD raise a warning but MAY continue with that embedding set to `null`.

### Unknown Fields

A reader MUST ignore unknown fields at any level. It MUST NOT raise an error for extra fields introduced by newer spec versions or custom extensions.

---

## 14. Conformance Levels

### Level 1 — Basic Reader

The implementation MUST:
- Open a `.udf` archive and parse `manifest.json`, `catalogue.json`, `content.json`
- Validate that `udf_version` begins with a supported major version
- Validate that all three `doc_id` values match
- Expose section titles and text

The implementation NEED NOT:
- Decode embeddings
- Handle `assets/`
- Handle `embeddings.bin`

### Level 2 — Semantic Reader

All Level 1 requirements, plus:
- Decode embeddings from both `embeddings.bin` (binary format) and `catalogue.json` (base64 format)
- Compute cosine similarity for semantic search
- Rank sections by relevance to a query embedding
- Lazy-load `embeddings.bin` (only when a search is performed)

### Level 3 — Intelligence Reader

All Level 2 requirements, plus:
- Expose intelligence fields: summaries, insights, key numbers, keywords
- Support multi-layer query routing (e.g., return `catalogue.summary` for broad queries)
- Read and expose DocMeta fields: `owner`, `department`, `tags`, `access_roles`

### Level 4 — Full Writer

All Level 3 requirements, plus:
- Produce valid `.udf` archives from source documents
- Populate all required fields in all three JSON files
- Support all four quantization modes
- Write `embeddings.bin` in binary format (preferred) or base64 in catalogue
- Write assets to `assets/` with correct references
- Apply smart compression: DEFLATE-9 for JSON, DEFLATE-1 for binary blobs, Store for images

---

## 15. Versioning and Forward Compatibility

UDF uses **semantic versioning** with two components: `MAJOR.MINOR`.

| Component | Meaning |
|---|---|
| MAJOR | Breaking changes to required fields or encoding |
| MINOR | New OPTIONAL fields; backward-compatible |

### Rules for Readers

- A reader that supports version `1.x` MUST be able to read all `1.y` archives for any `y`.
- A reader MUST reject archives with a higher major version than supported.
- A reader MUST ignore unrecognised fields (forward compatibility).

### Rules for Writers

- A writer for version `1.0` MUST produce archives where `udf_version` is exactly `"1.0"` unless producing a higher minor version.
- A writer MUST NOT omit any REQUIRED field.

### v1.0 → v1.1 Migration

v1.1 adds only OPTIONAL fields and one new OPTIONAL archive entry (`embeddings.bin`). All v1.0 archives are valid v1.1 archives. v1.1 writers that emit `embeddings.bin` SHOULD also set `manifest.embedding_format = "binary"` so v1.0 readers (which ignore unknown fields) still work correctly.

---

## 16. Extension Rules

Implementations MAY add custom fields anywhere in the three JSON files using the `"custom"` key defined in each schema. The value MUST be a JSON object.

```json
"custom": {
  "mylib_version": "2.1.0",
  "confidence": 0.92
}
```

Extension fields outside `"custom"` are also permitted (readers ignore unknown fields), but using `"custom"` is RECOMMENDED to avoid future naming conflicts.

Extensions MUST NOT redefine or conflict with any field defined in this specification.

---

## 17. Library Layer

A **library** is a `library.json` file that indexes multiple `.udf` archives in a directory, enabling cross-document search and management.

### library.json Schema

```json
{
  "library_version": "1.0",        // REQUIRED
  "created_at":      "<ISO-8601>", // REQUIRED
  "updated_at":      "<ISO-8601>", // REQUIRED
  "documents": [                   // REQUIRED — array of document entries
    {
      "doc_id":        "<string>",        // REQUIRED
      "title":         "<string>",        // REQUIRED
      "path":          "<string>",        // REQUIRED — absolute or relative path to .udf file
      "owner":         "<string>",        // OPTIONAL
      "department":    "<string>",        // OPTIONAL
      "tags":          ["<string>"],      // OPTIONAL
      "access_roles":  ["<string>"],      // OPTIONAL
      "section_count": <integer>,         // OPTIONAL
      "embedding_model": "<string>",      // OPTIONAL
      "keywords_bag":  ["<string>"],      // OPTIONAL — union of all section keywords
      "created_at":    "<ISO-8601>",      // OPTIONAL
      "added_at":      "<ISO-8601>"       // OPTIONAL — when the doc was added to the library
    }
  ]
}
```

### Library Rules

- `library.json` is typically stored in the same directory as the `.udf` archives it indexes.
- The `path` field MAY be relative to the directory containing `library.json`, or absolute.
- `keywords_bag` SHOULD be the union of all `keywords` arrays from all sections in the archive's `catalogue.json`. It enables fast keyword-based cross-document search without opening each archive.
- A document entry MUST be removed from `library.json` when the corresponding `.udf` file is deleted.
- `doc_id` values within a library MUST be unique.

### Cross-Document Search Algorithm

A library keyword search proceeds in two phases:

1. **Filter** (no archive I/O): scan `keywords_bag` across all entries; keep entries where at least one keyword matches the query tokens.
2. **Rank** (open matching archives): for matching archives, perform per-section semantic search; aggregate and return ranked results.

This two-phase approach avoids loading every archive into memory.
