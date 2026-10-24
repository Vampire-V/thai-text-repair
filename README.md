# thai-text-repair

> **Status: design in progress — no code yet.** Not published to PyPI, npm or crates.io.

Repair Thai text that was mangled on its way out of PDFs and legacy web pages, and tell you how much to trust the result.

ซ่อมข้อความภาษาไทยที่เพี้ยนจากการสกัด PDF/หน้าเว็บรุ่นเก่า และบอกว่าผลลัพธ์เชื่อได้แค่ไหน

## What it fixes

| Problem | Example (before → after) | Where it comes from |
|---|---|---|
| Microsoft Thai positional glyphs (PUA U+F700–F71A) | `การแขงขัน` → `การแข่งขัน` | Text layer of PDFs made with Windows Thai fonts (Angsana, TH Sarabun …) |
| Font-specific legacy codes (C1 0x80–0x9F) | `ผู\x8Cถูกกล\x8Bาวหา` → `ผู้ถูกกล่าวหา` | Old PDFs whose font maps tone marks to Latin-1 slots — applied **only** for fonts verified against real documents |
| Windows-1252 punctuation in HTML | `1 \x96 3` → `1 – 3` | Legacy government web pages |
| Decomposed sara am, duplicated/misordered marks, NBSP, zero-width, control chars | `คําสั่ง` → `คำสั่ง` | Many sources |

And when it can't fix something (e.g. a PDF font with no Unicode map, which yields `&2.@*\x0f@1…`), it says so: every result carries a quality verdict — `ok`, `garbled` or `needs_ocr`.

## Planned packages

One Rust core, with the mapping tables and test cases compiled in, published as:

- **Python** — PyPI `thai-text-repair` (PyO3 + maturin wheels)
- **JavaScript/TypeScript** — npm `thai-text-repair` (WebAssembly, Node and browser)
- **Rust** — crates.io `thai-text-repair`

See [docs/design-notes.md](docs/design-notes.md) for the agreed design and the evidence behind each rule.

## License

Code: dual-licensed under [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE), at your option.

Test data excerpts from the Royal Thai Gazette come from the [Open Law Data Thailand](https://huggingface.co/datasets/open-law-data-thailand/soc-ratchakitcha) dataset (CC-BY-4.0) and will be credited in the data files.
