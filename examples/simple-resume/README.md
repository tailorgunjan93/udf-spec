# Example: Simple Resume

This example shows a real `.udf` representation of Gunjan Tailor's résumé PDF, as produced by **docnest-ai 1.0** and lightly structured to demonstrate clean hierarchical section IDs.

---

## Document Characteristics

| Property | Value |
|---|---|
| Source | `GunjanTailor.pdf` (75.6 KB) |
| doc_id | `gunjan-tailor` |
| Sections | 8 (§1 root + §1.1–§1.7 children) |
| Embedding model | `huggingface/all-MiniLM-L6-v2` |
| Embedding dims | 384 |
| Quantization | `float16` |
| Embedding format | `binary` (stored in `embeddings.bin`) |
| Intelligence | Enabled (summaries + keywords per section) |
| Owner | Gunjan Tailor |
| Department | Engineering |
| Tags | resume, dotnet, csharp, azure, backend, test-automation |

---

## Archive Layout

```
gunjan-tailor.udf
├── manifest.json    # 487 B raw → ~305 B compressed
├── catalogue.json   # ~1.6 KB raw → ~500 B compressed  (no base64 embedding strings)
├── content.json     # ~3.1 KB raw → ~1.5 KB compressed
└── embeddings.bin   # 6.1 KB raw → ~6.0 KB compressed (8 × 768 B float16 embeddings)
```

**Total archive: ~8.9 KB** (12% of original 75.6 KB PDF).

`embeddings.bin` stores all 8 section embeddings as a flat binary blob (each 768 bytes for 384-dim float16). It is loaded lazily only when a search is performed — not at archive-open time.

---

## Section Tree

```
§1      Gunjan Tailor             (root — document identity)
├─ §1.1   Career Objective
├─ §1.2   Skills
├─ §1.3   Experience
├─ §1.4   Strengths
├─ §1.5   Achievements / Certifications
├─ §1.6   Education
└─ §1.7   Hobbies
```

All leaf sections (§1.1–§1.7) have full text in `content.json`.  
The root `§1` has `text: ""` — it is a structural container node.

---

## Query Routing Guide

| Query type | Best layer | Example |
|---|---|---|
| "Who is this person?" | `catalogue.summary` | Full document summary |
| "What skills does Gunjan have?" | `content.§1.2.text` | Raw skills text |
| "Which sections mention automation?" | `catalogue.section_index[*].keywords` | `§1.2`, `§1.3` |
| "Most relevant section to 'REST API'?" | semantic search on `embeddings.bin` | Cosine similarity ranking |
| "How many years of experience?" | `catalogue.key_numbers` | `8.5 years` |
| "What are the key highlights?" | `catalogue.insights` | 5 insight bullets |

---

## Real Content vs Auto-Generated

This example uses the **real extracted text** from `GunjanTailor.pdf` (not invented content).

Key facts verified from the actual PDF:

| Field | Correct value |
|---|---|
| Employer | Infogain India Pvt. Ltd. (project: Mitchell International) |
| Duration | Jul 2016 – Present |
| Education | B.Tech (IT), SKIT Jaipur, 2016, 74% |
| Award | Star Performer Award — Oct 2020; Spot Awards — Aug 2020, Jun 2023 |
| Hobbies | Reading motivational and self-help books |

---

## Difference from Auto-Generated UDF

The docnest-ai PDF parser produces **9 flat sections** (all at level 1) from this résumé because the PDF uses font-size headings without a clear two-level hierarchy. This example file shows the **ideal hierarchical representation** (§1 root + §1.1–§1.7 children) with LLM-enriched summaries and keywords.

To produce the hierarchical form automatically, run with full intelligence mode (without `--fast`):

```bash
docnest convert GunjanTailor.pdf \
  --owner "Gunjan Tailor" \
  --department "Engineering" \
  --tags "resume,dotnet,csharp,azure,backend"
```

---

## Notes

- `embeddings.bin` is the **preferred** embedding storage format (v1.1+). The per-section `"embedding"` field is **omitted** from `catalogue.json` when `embedding_format = "binary"`.
- The `doc_id` slug `"gunjan-tailor"` is derived by splitting the CamelCase filename (`GunjanTailor.pdf` → `gunjan-tailor`).
- Real embeddings would be 768-byte float16 blobs per section. This example ships without an `embeddings.bin` file — a conformant L2 reader would detect its absence and treat all embeddings as null.
