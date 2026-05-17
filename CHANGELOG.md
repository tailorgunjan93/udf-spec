# Changelog

All notable changes to the UDF specification are documented here.

This project adheres to [Semantic Versioning](https://semver.org/) (MAJOR.MINOR).

---

## [1.0] — 2026-05-17

Initial public release of the Universal Document Format specification.

### Defined

- ZIP-based container format with `.udf` extension
- Three required JSON files: `manifest.json`, `catalogue.json`, `content.json`
- Optional `assets/` folder for binary attachments
- `§id` hierarchical section identifier system
- Four quantization modes: `float32`, `float16`, `int8`, `binary`
- Base64 raw-bytes embedding encoding
- Intelligence fields: `summary`, `insights`, `key_numbers`, `keywords`
- Four conformance levels: L1 (Basic Reader) through L4 (Full Writer)
- Forward-compatibility rules for unknown fields
- Extension rules via `"custom"` key

### Reference Implementation

- [docnest-ai](https://github.com/tailorgunjan93/DOCNESTd) — Python 3.11+, Level 4 conformant

---

## [Unreleased]

### Planned for 1.1

- `library_mode` flag in `manifest.json` for multi-document collections
- Cross-document link references
- Streaming read support guidance
- Additional conformance test suite
