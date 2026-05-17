# UDF Implementations

This directory tracks known implementations of the UDF specification.

## Reference Implementation

| Library | Language | UDF version | Status | Link |
|---|---|---|---|---|
| **docnest-ai** | Python 3.11+ | 1.0 | ✅ Stable | [github.com/tailorgunjan93/DOCNESTd](https://github.com/tailorgunjan93/DOCNESTd) |

### docnest-ai features

- Reads and writes `.udf` v1.0 archives
- All four quantization modes: `float32`, `float16`, `int8`, `binary`
- Supports 10+ embedding providers via LangChain
- Supports 14+ LLM providers via LangChain
- Swappable storage backend (`ZipStorageBackend`, `DirectoryStorageBackend`)
- Five-layer query engine (L0–L4)
- CLI: `docnest convert`, `docnest query`, `docnest inspect`

```bash
pip install docnest-ai
docnest convert report.pdf        # → report.udf
docnest query report.udf "What is this about?"
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
        if "manifest.json" not in zf.namelist():
            raise ValueError("Invalid .udf — missing manifest.json")

        manifest  = json.loads(zf.read("manifest.json"))
        version   = manifest.get("udf_version", "?")
        if not version.startswith("1."):
            raise ValueError(f"Unsupported UDF version: {version}")

        catalogue = json.loads(zf.read("catalogue.json"))
        content   = json.loads(zf.read("content.json"))

    return {"manifest": manifest, "catalogue": catalogue, "content": content}


def get_section(udf: dict, section_id: str) -> dict | None:
    return udf["content"]["sections"].get(section_id)


def list_sections(udf: dict) -> list[dict]:
    return udf["catalogue"]["section_index"]


# Usage
udf = load_udf("report.udf")
print(udf["catalogue"]["summary"])

for section in list_sections(udf):
    print(f"{section['id']}  {section['title']}")

text = get_section(udf, "§1.2")
print(text["text"] if text else "Section not found")
```

## Minimal Writer (Python)

```python
import zipfile
import json
from datetime import datetime, timezone

def write_udf(output_path: str, manifest: dict, catalogue: dict, content: dict) -> None:
    with zipfile.ZipFile(output_path, "w", zipfile.ZIP_DEFLATED) as zf:
        zf.writestr("manifest.json",  json.dumps(manifest,  ensure_ascii=False, indent=2))
        zf.writestr("catalogue.json", json.dumps(catalogue, ensure_ascii=False, indent=2))
        zf.writestr("content.json",   json.dumps(content,   ensure_ascii=False, indent=2))


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
    "intelligence": False
}

catalogue = {
    "doc_id": "my-document",
    "title": "My Document",
    "source": "/path/to/my-document.pdf",
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
