# Design notes (2026-10-24)

สถานะ: brainstorming — ตกลงแล้ว **ส่วนที่ 1 (API)** · ส่วนที่ 2–4 ยังไม่ได้ออกแบบ · ยังไม่มีโค้ด

ที่มา: แยกออกจากโปรเจกต์ค้นกฎหมาย (mcp_legal) หลังทดลองดึงคำวินิจฉัย/ข้อหารือจากเว็บหน่วยงานรัฐแล้วพบว่า
ข้อความที่สกัดได้มีอักขระเพี้ยนหลายแบบที่ไลบรารีที่มีอยู่ไม่ได้แก้

## การตัดสินใจที่ตกลงแล้ว

| เรื่อง | ตัดสิน | เหตุผล |
|---|---|---|
| เผยแพร่ | สาธารณะตั้งแต่แรก | |
| โครงสร้าง | แกน Rust ตัวเดียว + ฝังข้อมูล (`data/`) ตอนคอมไพล์ → Python (PyO3/maturin) + npm (wasm-bindgen) + crates.io | กฎและข้อมูลมีชุดเดียว — สองภาษาได้ผลตรงกันโดยโครงสร้าง ไม่ต้องดูแลสองชุด |
| ชื่อ | `thai-text-repair` ทุกที่ | ตรวจ 24 ต.ค. 2569: ว่างบน PyPI, npm, crates.io (ยังไม่ได้จอง — ชื่อจะถูกจองตอน publish ครั้งแรก) |
| ไลเซนส์โค้ด | MIT OR Apache-2.0 | ธรรมเนียม Rust · เข้ากันกับ pythainlp (Apache-2.0) |

## ส่วนที่ 1: API (ตกลงแล้ว)

```python
r = repair(text, origin="pdf" | "html" | "plain", fonts=[...], explain=False)
r.text      # ข้อความหลังซ่อม
r.quality   # verdict: ok | garbled | needs_ocr · tone_density · thai_space_per_1k · leftover · reasons
r.changes   # explain=True: [Change(start, end, before, after, rule)]
q = assess(text)   # วัดคุณภาพอย่างเดียว
```
- `origin` จำเป็นเพราะรหัสเดียวกันแปลต่างกันตามต้นทาง (`\x96` ใน HTML cp1252 = "–" แต่ใน PDF ฟอนต์เก่าเป็นวรรณยุกต์)
- `fonts` ใช้กับตาราง C1 เท่านั้น — ฟอนต์ที่ไม่อยู่ในรายการยืนยัน → **ไม่เดา** นับเป็น leftover → `needs_ocr`
- ลำดับกฎตายตัว ทุกการเปลี่ยนอ้าง rule id ได้:
  `ms_thai_pua → c1_font:<font> → cp1252_punct (html) → symbol_pua (→ • + ธง) → nfc → sara_am → mark_order → dup_mark → nbsp → zero_width → control → form_feed → whitespace`
- ไม่ทำ: สกัด PDF เอง (แนะนำ `pdftotext -raw` + `pdffonts` ใน README) · ตัดคำ · แก้คำสะกดผิดของต้นทาง

## หลักฐานที่มีแล้ว (spike 24 ต.ค. 2569)

ชุดทดลอง: PDF ราชกิจจาฯ 439 ฉบับ (2006–2024, Open Law Data Thailand บน HF, CC-BY-4.0) + PDF/HTML จากเว็บหน่วยงานอีก ~650 ฉบับ

| ชนิด | พบ | การยืนยัน | สรุป |
|---|---|---|---|
| Microsoft Thai PUA U+F700–F71A (21 รหัสที่พบ) | ~91% ของ PDF ราชกิจจาฯ | ตาราง HarfBuzz `hb-ot-shape-complex-thai` · OCR อิสระ (iapp) ตรง เช่น ้×1,028 ่×555 ์×328 ็×88 · โหวตด้วยพจนานุกรม pythainlp ตรงทุกรหัสที่ตัดสินได้ | ใช้ได้ทุกฟอนต์ |
| C1 0x80–0x9F ของฟอนต์ DBHeavent (WinAnsi subset) | 1 ใน ~544 PDF | พจนานุกรม: 0x8B ่ (57) · 0x8C ้ (120) · 0x8F ์ (137) · 0x9A ็ (186) · 0x98 ั (4) — อีก 5 รหัสหายาก (0x81 0x82 0x84 0x86 0x87) ตัดสินไม่ได้ · **ไม่ตรง** Apple Mac Thai (THAI.TXT) และ Windows-874 ไม่มีวรรณยุกต์ช่วงนี้ | ตารางเฉพาะฟอนต์ ผูกด้วยชื่อฟอนต์ |
| Symbol/Wingdings PUA U+F0xx | ~6 ฉบับ | OCR บอก • ○ บางตัว | แทน • + ติดธง |
| ฟอนต์ไม่มี ToUnicode (เช่น Helvetica [no-uni]) | 7 ฉบับ (2024) | — | กู้ไม่ได้ → ตรวจจับ → `needs_ocr` |
| cp1252 “ ” – ใน HTML | หน้าเว็บกรมสรรพากรรุ่นเก่า | บริบท | เฉพาะ `origin="html"` |
| ํา แยก · NBSP · ช่องว่างแทรกกลางคำ · \f | ทุกแหล่ง | pythainlp `normalize` แก้ ํา/วรรณยุกต์ซ้ำ/ลำดับ/zero-width ได้ แต่ไม่แก้ PUA, C1, NBSP, cp1252 | ทำเองในแกน Rust |

ตัววัดคุณภาพที่ใช้ได้จริง: ความหนาแน่นไม้เอก+โท ต่ออักษรไทย (ปกติ ~5–10%) · ช่องว่างไทย-ไทยต่อพันตัว (HTML ปกติ ~18–26) · อักขระนอกชุดปกติที่เหลือ.
ตัววัดที่**ใช้ไม่ได้**: อัตราคำนอกพจนานุกรม (แยกฉบับเพี้ยนกับฉบับดีไม่ออก).

## ข้อจำกัด/งานค้างที่รู้แล้ว

- C1 ยืนยันได้ฟอนต์เดียว 5 รหัส — อีก 5 รหัสต้องหาเอกสารเพิ่ม
- การโหวตด้วยพจนานุกรมเอียงไปทางคำพบบ่อย: ๊ ๋ ในชื่อเฉพาะ/คำทับศัพท์ (เช่น "เป๊ปซี่") ต้องพึ่งตารางมาตรฐาน
- OCR iapp มีที่ผิดเอง และบางฉบับผูกผิดเอกสาร — ใช้เป็นหลักฐานแบบโหวตเสียงข้างมากเท่านั้น
- กฎ `mark_order` ยังไม่ได้ทดสอบกับข้อมูลจริง
- **ข้อมูลทดสอบใน repo สาธารณะ**: ใช้ได้เฉพาะข้อความจากราชกิจจาฯ (CC-BY-4.0 + เครดิต) และข้อความสังเคราะห์ ·
  ห้ามใส่ข้อความจากเว็บที่ข้อกำหนดห้ามทำซ้ำ (ศาลยุติธรรม, กรมสรรพากร, กขค., กกต., ศาลรัฐธรรมนูญ — ข้อ 3.3/3.5) — case จากแหล่งเหล่านั้นต้องเขียนใหม่เป็นตัวอย่างสังเคราะห์ที่มีรหัสเพี้ยนแบบเดียวกัน

## ยังไม่ได้ออกแบบ

- ส่วนที่ 2: รูปแบบไฟล์ใน `data/` (ตาราง, แคตตาล็อก case, หลักฐาน, เครดิต)
- ส่วนที่ 3: โครง repo / build / CI (maturin wheels หลาย OS, wasm-pack) / การ publish
- ส่วนที่ 4: การทดสอบ (case conformance ทุกภาษา, ตัวอธิบาย diff: ทุกจุดที่เปลี่ยนต้องมีกฎรองรับ)

## อ้างอิง

- HarfBuzz Thai shaper (Windows/Mac PUA): https://skia.googlesource.com/third_party/harfbuzz/+/0.9.24/src/hb-ot-shape-complex-thai.cc
- Apple Thai mapping: https://www.unicode.org/Public/MAPPINGS/VENDORS/APPLE/THAI.TXT
- windows-874: https://iana.org/assignments/charset-reg/windows-874
- ตาราง PUA ในโปรเจกต์ constitution: https://github.com/bact/constitution
- `@thai/text-cleaner` (JS, ทับบางส่วนกับกฎมาตรฐาน): https://jsr.io/@thai/text-cleaner
- ชุดข้อมูลราชกิจจาฯ: https://huggingface.co/datasets/open-law-data-thailand/soc-ratchakitcha
