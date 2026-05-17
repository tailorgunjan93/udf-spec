# Example: Simple Resume (PDF)

A minimal UDF example generated from a single-page PDF resume.

## Characteristics

- **Source format:** PDF (text-native, no OCR needed)
- **Parser:** PyMuPDFParser (font-size heading detection)
- **Sections:** 8 (1 title + 7 content sections)
- **Tables:** 0
- **Images:** 0
- **Intelligence:** Enabled (LLM summaries + keywords)
- **Embedding model:** `huggingface/all-MiniLM-L6-v2` (384 dims)
- **Quantization:** `float16`

## Section Tree

```
§1       Gunjan Tailor           (level 1 — document title)
  §1.1   Career Objective        (level 2)
  §1.2   Skills                  (level 2)
  §1.3   Experience              (level 2)
  §1.4   Strengths               (level 2)
  §1.5   Achievements / Certs    (level 2)
  §1.6   Education               (level 2)
  §1.7   Hobbies                 (level 2)
```

## Query examples

| Query | Layer | Answer source |
|---|---|---|
| "What is this document about?" | L0 | `catalogue.summary` |
| "Summarise" | L0 | `catalogue.summary` |
| "What skills does this person have?" | L2 | `content.sections["§1.2"].text` via LLM |
| "What is the education background?" | L1 | `catalogue.section_index["§1.6"].summary` |

## Files

| File | Size | Description |
|---|---|---|
| `manifest.json` | ~400 B | Format metadata |
| `catalogue.json` | ~3 KB | Section index + embeddings |
| `content.json` | ~1.5 KB | Full section text |

## Notes

The `embedding` values in `catalogue.json` are intentionally short placeholders (not real 384-dim vectors). In a real `.udf` file produced by docnest-ai, each `embedding` field would be a Base64 string approximately 1024 characters long (384 dims × 2 bytes/dim = 768 bytes → ~1024 chars Base64).
