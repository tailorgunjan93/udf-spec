# UDF Implementations

This directory tracks known implementations of the UDF specification.

## Reference Implementation

| Library | Language | UDF version | Conformance | Status | Link |
|---|---|---|---|---|---|
| **docnest-ai** | Python 3.11+ | 1.0 | L4 | ✅ Stable | [github.com/tailorgunjan93/DOCNESTd](https://github.com/tailorgunjan93/DOCNESTd) |

### docnest-ai features (v1.0)

**Reading & writing**
- Reads and writes `.udf` v1.0 archives (Level 4 conformant)
- All four quantization modes: `float32`, `float16`, `int8`, `binary`
- Swappable storage backends: `ZipStorageBackend` (default), `DirectoryStorageBackend` (debug)
- Compact JSON output (`separators=(',', ':')`) — 15–20% smaller than pretty-printed

**Embedding storage (v1.1 features)**
- `embeddings.bin` binary blob format — ~87% smaller `catalogue.json` vs base64
- Lazy embedding loading: matrix is only decoded when a search is triggered
- Smart ZIP compression: DEFLATE-9 for JSON, DEFLATE-1 for binary, Store for images
- Backward-compatible: auto-detects binary vs base64 format at read time

**AI pipeline**
- 10+ embedding providers via LangChain (HuggingFace, OpenAI, Cohere, etc.)
- 14+ LLM providers via LangChain for intelligence enrichment
- Five-layer query engine (L0 catalogue → L4 full-text LLM)
- Fast mode (`--fast`): embeddings without LLM; skips keywords/summaries

**Organisational metadata (DocMeta)**
- `owner`, `department`, `tags`, `access_roles`, `version`, `last_updated` fields
- Written into `manifest.json` at convert time via CLI flags

**Library layer**
- `library.json` multi-document index for cross-document search
- `keywords_bag` field for fast pre-filter without opening each archive
- `docnest library init/add/list/search/remove` CLI commands

**HTML viewer**
- `docnest view <file.udf>` generates a self-contained HTML page
- Sidebar table of contents with Intersection Observer scroll-sync
- Section heading hierarchy, table rendering, keyword badges
- Metadata bar: owner, department, model, quantization, date

**CLI**
```bash
# Convert
docnest convert report.pdf --owner "Alice" --department "Finance" --tags "q4,2024"

# Search
docnest query report.udf "What is the revenue forecast?"

# Inspect
docnest inspect report.udf

# View as HTML (opens in browser)
docnest view report.udf

# Library management
docnest library init ./docs/
docnest library add  ./docs/ report.udf
docnest library list ./docs/
docnest library search ./docs/ "revenue forecast"
```

---

## In-Progress Implementations

| Library | Language | UDF version | Status | Link |
|---|---|---|---|---|
| udf-reader-vscode | TypeScript | 1.0 | 🔨 In progress | [github.com/tailorgunjan93/udf-reader-vscode](https://github.com/tailorgunjan93/udf-reader-vscode) |
| synapse-local | Rust (Tauri) | 1.0 | 📋 Planned | [github.com/tailorgunjan93/synapse-local](https://github.com/tailorgunjan93/synapse-local) |

---

## Add Your Implementation

If you've built a reader, writer, or tool that works with `.udf` files:

1. Fork this repository
2. Add your entry to the table above
3. Open a pull request

Please include:
- Library name and link
- Language / platform
- Which UDF version it targets
- Conformance level (see [SPEC.md §14](../SPEC.md#14-conformance-levels))
- Any known limitations

---

## Minimal Reader (Python)

A Level 1 conformant reader requires only Python stdlib:

```python
import zipfile
import json

def load_udf(path: str) -> dict:
    with zipfile.ZipFile(path, "r") as zf:
        names = zf.namelist()
        for required in ("manifest.json", "catalogue.json", "content.json"):
            if required not in names:
                raise ValueError(f"Invalid .udf — missing {required}")

        manifest  = json.loads(zf.read("manifest.json"))
        version   = manifest.get("udf_version", "?")
        if not version.startswith("1."):
            raise ValueError(f"Unsupported UDF version: {version}")

        catalogue = json.loads(zf.read("catalogue.json"))
        content   = json.loads(zf.read("content.json"))

        for key in ("doc_id",):
            vals = {manifest[key], catalogue[key], content[key]}
            if len(vals) != 1:
                raise ValueError(f"doc_id mismatch across manifest/catalogue/content")

    return {"manifest": manifest, "catalogue": catalogue, "content": content}


def get_section(udf: dict, section_id: str) -> dict | None:
    return udf["content"]["sections"].get(section_id)


def list_sections(udf: dict) -> list[dict]:
    return udf["catalogue"]["section_index"]


# Usage
udf = load_udf("report.udf")
print(udf["catalogue"].get("summary", ""))

for section in list_sections(udf):
    print(f"{section['id']}  {section['title']}")

text = get_section(udf, "§1.2")
print(text["text"] if text else "Section not found")
```

---

## Level 2 Reader — Binary Embeddings (Python)

A Level 2 reader that supports the v1.1 `embeddings.bin` format:

```python
import zipfile, json, math
import numpy as np

DTYPE = {"float32": "float32", "float16": "float16", "int8": "int8", "binary": "uint8"}

def stride(dims: int, quantization: str) -> int:
    if quantization == "binary":
        return math.ceil(dims / 8)
    return dims * {"float32": 4, "float16": 2, "int8": 1}[quantization]

def load_embeddings(path: str) -> np.ndarray | None:
    with zipfile.ZipFile(path, "r") as zf:
        manifest = json.loads(zf.read("manifest.json"))
        catalogue = json.loads(zf.read("catalogue.json"))

        fmt   = manifest.get("embedding_format", "base64")
        dims  = manifest.get("embedding_dims") or 0
        quant = manifest.get("quantization") or "float16"
        n     = len(catalogue["section_index"])

        if fmt == "binary" and "embeddings.bin" in zf.namelist():
            raw = zf.read("embeddings.bin")
            s   = stride(dims, quant)
            if len(raw) != n * s:
                return None   # wrong size — treat as missing
            mat = np.frombuffer(raw, dtype=DTYPE[quant]).reshape(n, -1).astype(np.float32)
            return mat

        # Fallback: base64 per-section
        rows = []
        for entry in catalogue["section_index"]:
            b64 = entry.get("embedding")
            if b64:
                import base64
                rows.append(np.frombuffer(base64.b64decode(b64), dtype=DTYPE[quant]).astype(np.float32))
            else:
                rows.append(np.zeros(dims, dtype=np.float32))
        return np.stack(rows) if rows else None
```

---

## Minimal Writer (Python)

```python
import zipfile, json
from datetime import datetime, timezone

def write_udf(output_path: str, manifest: dict, catalogue: dict, content: dict) -> None:
    with zipfile.ZipFile(output_path, "w") as zf:
        # JSON: max compression (repetitive keys compress very well)
        for name, obj in [("manifest.json", manifest), ("catalogue.json", catalogue), ("content.json", content)]:
            data = json.dumps(obj, ensure_ascii=False, separators=(",", ":")).encode("utf-8")
            zf.writestr(name, data, compress_type=zipfile.ZIP_DEFLATED, compresslevel=9)


# Minimal manifest
manifest = {
    "udf_version": "1.0",
    "doc_id": "my-document",
    "title": "My Document",
    "source_format": "pdf",
    "created_at": datetime.now(timezone.utc).isoformat(),
    "embedding_model": "huggingface/all-MiniLM-L6-v2",
    "embedding_dims": 384,
    "quantization": "float16",
    "section_count": 1,
    "intelligence": False,
    "embedding_format": "base64"
}

catalogue = {
    "doc_id": "my-document",
    "title": "My Document",
    "summary": "",
    "insights": [],
    "key_numbers": [],
    "section_index": [
        {
            "id": "§1",
            "title": "Introduction",
            "level": 1,
            "parent_id": None,
            "children": [],
            "summary": "",
            "keywords": ["introduction"],
            "token_count": 42,
            "embedding": None
        }
    ],
    "embedding_model": "huggingface/all-MiniLM-L6-v2",
    "embedding_dims": 384,
    "quantization": "float16"
}

content = {
    "doc_id": "my-document",
    "sections": {
        "§1": {
            "title": "Introduction",
            "level": 1,
            "text": "This is the introduction section.",
            "tables": [],
            "images": []
        }
    }
}

write_udf("my-document.udf", manifest, catalogue, content)
```
