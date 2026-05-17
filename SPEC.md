# UDF Specification — Version 1.0

**Status:** Stable  
**Published:** 2026-05-17  
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
| **Reader** | Software that opens and queries `.udf` files |
| **Writer** | Software that produces `.udf` files |

The key words MUST, MUST NOT, REQUIRED, SHOULD, SHOULD NOT, MAY, and OPTIONAL in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## 2. Design Goals

1. **Zero-dependency readability** — The three core JSON files MUST be parseable with standard library only (Python `json`, Node.js `JSON.parse`, etc.).
2. **Single-file portability** — One `.udf` file contains everything needed to query a document offline.
3. **Swappable AI** — The format records which model produced the embeddings but does not mandate a specific provider.
4. **Forward-compatible** — New fields MUST be ignored by readers that do not recognise them.
5. **Human-inspectable** — A developer can `unzip -p report.udf manifest.json | python -m json.tool` and read the output.

---

## 3. Container Format

A `.udf` file is a **ZIP archive** (PKZIP format, version 2.0 or higher).

- Compression method: DEFLATE (method 8) is RECOMMENDED. Store (method 0) is permitted.
- Encoding: All JSON entry names and content MUST be UTF-8 encoded.
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
└── assets/             OPTIONAL
    ├── img_0001.png
    ├── img_0002.jpg
    └── attachment.xlsx
```

### Rules

- `manifest.json`, `catalogue.json`, and `content.json` MUST appear at the root level of the archive (no subdirectory prefix).
- The `assets/` folder is OPTIONAL. Files inside it MAY use any naming convention.
- A reader that encounters unknown files at the root level MUST ignore them without error.
- Future versions of this spec MAY define additional root-level files.

---

## 5. manifest.json Schema

`manifest.json` describes the archive itself and the embedding configuration.

### Full Schema

```json
{
  "udf_version":      "<string>",        // REQUIRED — e.g. "1.0"
  "doc_id":           "<string>",        // REQUIRED — unique identifier
  "title":            "<string>",        // REQUIRED — human-readable title
  "source_format":    "<string>",        // REQUIRED — "pdf", "docx", "txt", "html", "md", …
  "created_at":       "<ISO-8601>",      // REQUIRED — UTC timestamp
  "embedding_model":  "<string|null>",   // REQUIRED — e.g. "huggingface/all-MiniLM-L6-v2" or null
  "embedding_dims":   <integer|null>,    // REQUIRED — number of dimensions, or null if no embeddings
  "quantization":     "<string|null>",   // REQUIRED — "float32"|"float16"|"int8"|"binary"|null
  "section_count":    <integer>,         // REQUIRED — total number of sections
  "intelligence":     <boolean>,         // REQUIRED — true if AI intelligence fields are populated
  "language":         "<string>",        // OPTIONAL — BCP-47 language tag, e.g. "en", "fr"
  "source_path":      "<string>",        // OPTIONAL — original file path at conversion time
  "producer":         "<string>",        // OPTIONAL — name + version of the software that wrote this file
  "custom":           <object>           // OPTIONAL — arbitrary extension data
}
```

### Field Constraints

| Field | Constraint |
|---|---|
| `udf_version` | MUST match `^\d+\.\d+$`. Readers MUST reject archives where the major version is higher than supported. |
| `doc_id` | MUST be non-empty. SHOULD be URL-safe (alphanumeric, `-`, `_`). MUST match `doc_id` in `catalogue.json` and `content.json`. |
| `created_at` | MUST be a valid ISO 8601 datetime with timezone. |
| `quantization` | MUST be one of `"float32"`, `"float16"`, `"int8"`, `"binary"`, or `null`. |
| `embedding_dims` | MUST be a positive integer when `embedding_model` is non-null. |
| `section_count` | MUST equal the number of entries in `catalogue.json → section_index`. |

---

## 6. catalogue.json Schema

`catalogue.json` is the document's index. It contains per-section metadata and embeddings, plus document-level intelligence.

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
      "embedding":  "<base64>|null" // OPTIONAL — encoded vector (see §9)
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
- The order of entries in `section_index` SHOULD reflect document reading order.
- `id` values MUST be unique within the archive.
- `parent_id` MUST reference an `id` that appears elsewhere in `section_index`, or be `null`.
- `children` MUST list exactly the `id` values whose `parent_id` points to this section. An empty array `[]` is valid.
- If `embedding` is non-null, its decoded byte length MUST equal `embedding_dims × bytes_per_element` (see §10).

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

Embeddings are stored in `catalogue.json → section_index[*].embedding` as **Base64-encoded raw bytes** (standard alphabet, no line wrapping, with or without padding).

### Encoding Process

1. Obtain the float vector (list of `embedding_dims` floats).
2. Quantize according to the `quantization` mode (see §10).
3. Encode the resulting byte array using Base64.
4. Store the Base64 string in `"embedding"`.

### Decoding Process

1. Decode Base64 to bytes.
2. Interpret bytes according to `manifest.quantization` and `manifest.embedding_dims`.
3. For `int8` and `binary`, dequantize to recover approximate float values.

If `embedding` is `null`, it means the section was not embedded (acceptable for title-only nodes or when embedding was skipped).

---

## 10. Quantization Modes

### float32

- NumPy dtype: `float32` (IEEE 754 single precision)
- Bytes per dimension: 4
- Total bytes: `embedding_dims × 4`
- Endianness: little-endian

### float16

- NumPy dtype: `float16` (IEEE 754 half precision)
- Bytes per dimension: 2
- Total bytes: `embedding_dims × 2`
- Endianness: little-endian
- **Default mode** for docnest-ai 1.0

### int8

- NumPy dtype: `int8`
- Bytes per dimension: 1
- Total bytes: `embedding_dims × 1`
- Encoding: scale each float `v` to `round(v × 127)`, clamped to `[-127, 127]`
- Decoding: divide each int8 value by `127.0`

### binary

- NumPy dtype: `uint8` bit-packed
- Bytes per dimension: 1/8
- Total bytes: `ceil(embedding_dims / 8)`
- Encoding: threshold each float at 0; bit `i` = 1 if `v[i] > 0`, else 0. Pack bits MSB-first.
- Decoding: unpack bits; interpret 1 as `+1.0`, 0 as `-1.0` (or `0.0` — implementation choice).

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

### Corrupt Embeddings

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

### Level 2 — Semantic Reader

All Level 1 requirements, plus:
- Decode embeddings for all supported quantization modes
- Compute cosine similarity for semantic search
- Rank sections by relevance to a query embedding

### Level 3 — Intelligence Reader

All Level 2 requirements, plus:
- Expose intelligence fields: summaries, insights, key numbers, keywords
- Support multi-layer query routing (e.g., return `catalogue.summary` for broad queries)

### Level 4 — Full Writer

All Level 3 requirements, plus:
- Produce valid `.udf` archives from source documents
- Populate all required fields in all three JSON files
- Support all four quantization modes
- Write assets to `assets/` with correct references

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
