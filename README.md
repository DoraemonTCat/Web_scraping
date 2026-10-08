# News Crawler — ชั้นดึงเว็บ

ดึงข่าวจากแหล่งที่ตั้งไว้ แล้วคืนข้อความที่ล้างแล้วเป็น JSON
**ไม่แตะฐานข้อมูล ไม่นับ keyword ไม่มี framework** — สองอย่างหลังเป็นงานฝั่ง backend

ตรงตามคู่มือ *News Crawler สำหรับ Junior Dev* ขั้น 1–2 (คู่มือหน้า 3)

---

## รันยังไง

```bash
pip install -r requirements.txt
```

```bash
python cli.py
```

ดึงทุกแหล่งที่สถานะ `ready` (19 แหล่ง) แล้วเขียน `output/crawl_<เวลา>.json`

ตัวเลือกอื่น:

```bash
python cli.py --max-articles 10
```

```bash
python cli.py --only "US FDA"
```

```bash
python cli.py --status all
```

เทสต์ (ไม่ยิงเว็บจริง) — รันได้เลยไม่ต้องมี pytest:

```bash
python test_crawler.py
```

ถ้าติดตั้ง pytest ไว้แล้วจะใช้แบบนี้ก็ได้:

```bash
python -m pytest test_crawler.py -v
```

---

## ผลรันล่าสุด

`python cli.py` (ไม่จำกัดจำนวน) เมื่อ 8 ต.ค. 2026

| | |
|---|---|
| แหล่งข่าว | 19 สำเร็จ · 0 ถูกบล็อก · 0 ผิดพลาด |
| บทความ | 384 ชิ้น |
| มีเนื้อเต็ม | 252 (`text_source: article`) |
| มีแค่หัวข้อ+ย่อ | 112 (`rss_summary`) |
| ข้ามเพราะเป็นไฟล์เอกสาร | 1 (`skipped_document`) |
| ไม่มีข้อความเลย | 19 (`none`) |
| ดึงเนื้อไม่สำเร็จ | 0 |

**ตรวจสอบความถูกต้องแล้ว** สุ่ม 7 บทความจาก 7 แหล่ง ดึงสดมาเทียบ — `text` ตรงกันเป๊ะทุกตัว (ต่าง 0 ตัวอักษร)
และสุ่มข้อความจากกลางบทความไปหาในหน้าจริง เจอครบ

112 ชิ้นที่มีแค่หัวข้อคือเว็บข่าวเชิงพาณิชย์ ซึ่งตั้งใจไม่ดึงเนื้อเต็ม ดูเหตุผลใน [SURVEY.md](SURVEY.md)
ส่วน 19 ชิ้นที่ไม่มีข้อความเลยคือรายการที่ feed ต้นทางส่ง `<title/>` และ `<description/>` ว่างมาเอง
(CNN 2 ชิ้นเป็นคลิปวิดีโอ, drugfree.org 17 ชิ้นเป็นหน้าแนะนำองค์กร)

---

## ไฟล์ไหนทำอะไร

| ไฟล์ | หน้าที่ |
|---|---|
| `crawler.py` | **หัวใจ** — `Fetcher`, `extract_links`, `extract_article`, `parse_feed`, `crawl_source` |

| `normalize.py` | `normalize_text` — ล้างข้อความให้แมพ keyword ได้ถูก |
| `runner.py` | `run()` — ดึงทุกแหล่งพร้อมกัน รวมผลเป็น dict เดียว |
| `cli.py` | เปลือกบรรทัดคำสั่ง เขียนไฟล์ JSON |
| `net.py` | `ssl_context()` — ตัวช่วยเรื่องใบรับรอง HTTPS |
| `sources.json` | แหล่งข่าว 31 รายการ พร้อมสถานะจากผลสำรวจ |
| `test_crawler.py` | เทสต์ด้วย `FakeFetcher` ไม่ยิงเว็บจริง |
| `SURVEY.md` | ผลสำรวจแหล่งข่าวทั้ง 31 (เฟส 0) |

ไฟล์ `survey.py`, `probe_*.py` เป็นเครื่องมือสำรวจของเฟส 0 ไม่ใช่ส่วนของ crawler

---

## ฝั่ง backend เอาไปต่อยังไง

ไม่มีไฟล์ไหน `import django` เลย วางใน `workers/` แล้วเรียกได้ทันที

**ทาง A — เรียกตรง (แนะนำ)**

```python
from runner import run, load_sources

_, sources = load_sources()
report = run(sources)          # ได้ dict ไม่ผ่านไฟล์
```

**ทาง B — อ่านไฟล์ JSON**

```bash
python cli.py
```

แล้วอ่าน `output/crawl_*.json`

**เรียกทีละแหล่ง** (เช่นเมื่ออยากคุม transaction เอง)

```python
from crawler import Fetcher, crawl_source

with Fetcher() as fetcher:
    result = crawl_source(source_dict, fetcher)
```

**ยกเลิกกลางรอบ** (คู่มือหน้า 4)

```python
from runner import Canceller, run

c = Canceller()
# ... อีกเธรดเรียก c.cancel() ...
report = run(sources, canceller=c)
```

`crawler` จะเช็กก่อนดึงบทความแต่ละชิ้น จึงหยุดได้โดยไม่ต้องรอจนจบ

---

## หน้าตาผลลัพธ์

```json
{
  "run_at": "2026-10-08T13:09:00+07:00",
  "normalized": true,
  "status": "SUCCESS",
  "cancelled": false,
  "summary": {
    "sources_total": 19, "ok": 19, "blocked": 0, "error": 0,
    "articles_total": 186, "articles_with_full_text": 126
  },
  "sources": [{
    "name": "US FDA - Recalls & Alerts",
    "url": "https://www.fda.gov/.../recalls/rss.xml",
    "kind": "rss",
    "group": "regulator",
    "status": "ok",
    "error": null,
    "cancelled": false,
    "articles": [{
      "url": "https://www.fda.gov/safety/recalls.../pacific-health-sciences...",
      "title": "Pacific Health Sciences Voluntarily Recalls One Lot of...",
      "published_at": "2026-10-06T13:09:00-04:00",
      "summary": "Pacific Health Sciences is voluntarily recalling one lot...",
      "text": "Summary Company Announcement Date: October 06, 2026 ...",
      "text_source": "article",
      "fetched_at": "2026-10-08T13:09:12+07:00",
      "error": null
    }]
  }]
}
```

### ช่องที่ต้องรู้

**`text_source`** บอกว่าข้อความมาจากไหน — ใช้ค่านี้ตีความผลการนับ keyword

| ค่า | หมายถึง |
|---|---|
| `article` | เนื้อข่าวเต็ม (เว็บหน่วยงานรัฐ) |
| `rss_summary` | แค่หัวข้อกับย่อข่าว (เว็บข่าวเชิงพาณิชย์) |
| `skipped_document` | ลิงก์ชี้ไปไฟล์ PDF/Word ไม่ได้ดึง — ดู `error` |
| `none` | ไม่มีข้อความเลย feed ต้นทางส่งมาว่าง |

ถ้าไม่ดูช่องนี้ จะงงว่าทำไมข่าว BBC เจอ keyword น้อยกว่าข่าว FDA — เพราะข้อความสั้นกว่ามาก ไม่ใช่เพราะเนื้อหาไม่เกี่ยว

**`status` ของแต่ละแหล่ง** มี 3 ค่า

| ค่า | หมายถึง |
|---|---|
| `ok` | ดึงสำเร็จ |
| `blocked` | เว็บปฏิเสธ (`robots.txt` ห้าม หรือ HTTP 401/403/429) — **ไม่ใช่ระบบพัง** |
| `error` | ผิดพลาดจริง เช่น timeout |

รอบไหนมีอย่างน้อยหนึ่งแหล่งสำเร็จ = `SUCCESS` ไม่มีเลย = `FAILED` (คู่มือหน้า 3 ขั้น 4)

**`normalized: true`** — `title`, `summary`, `text` ล้างมาแล้วทั้งหมด เอาไปแมพ keyword ได้เลย ไม่ต้องล้างซ้ำ

---

## การหาลิงก์บทความ

`extract_links()` ค้น 2 ที่ (คู่มือหน้า 3 ขั้น 1):

1. **`<a href>`** — ลิงก์ปกติในหน้า
2. **`<script>` / JSON** — บางเว็บโหลดรายการข่าวผ่าน JavaScript ลิงก์จึงไม่อยู่ใน `<a href>` แต่ฝังเป็นข้อความใน JSON ที่ติดมากับหน้า

แต่ละลิงก์มีช่อง `found_in` บอกว่ามาจาก `anchor` หรือ `script`

ตัวกรองที่ใส่ไว้: ตัดไฟล์ประกอบหน้าเว็บทิ้ง (`.js`, `.css`, รูป, ฟอนต์) และตัดโฟลเดอร์อย่าง
`/wp-content/`, `/static/`, `/_next/` เพราะเว็บที่ไม่ได้ตั้ง `link_pattern` จะได้ขยะพวกนี้ปนมา

> **สถานะปัจจุบัน:** ค้น `<script>` แล้ว **ยังไม่ได้ลิงก์เพิ่มจาก 13 แหล่ง HTML ที่ใช้อยู่** (429 ลิงก์เท่าเดิม)
> เพราะเว็บที่โหลดข่าวด้วย JavaScript จริง ๆ อย่าง BBC เราใช้ RSS แทนไปแล้ว ส่วน EMA ตรวจแล้วพบว่า
> หน้าแรกมีลิงก์ข่าวแค่ 6 ลิงก์จริง ไม่ได้ซ่อนไว้ในสคริปต์
>
> ฟีเจอร์นี้จึงเป็นตาข่ายรองรับสำหรับแหล่งข่าวที่จะเพิ่มเข้ามาทีหลัง มีเทสต์ยืนยันว่าทำงานได้จริง

## เรื่อง normalize ที่ต้องรู้

`normalize_text()` ทำ 4 อย่าง:

1. **NFC** — รวมสระ/วรรณยุกต์ที่แยกส่วน
2. **รวม "ํา" กับ "ำ"** — NFC ทำให้ไม่ได้ Unicode ไม่ถือว่าเทียบเท่า ต้องแทนที่เอง
3. **ลบ zero-width space** — ไม่ใช่แปลงเป็นช่องว่าง
4. **ยุบช่องว่างซ้ำ**

> **ต่างจาก `workers/normalizer.py` ตรงข้อ 3** ของเดิมแปลง zero-width space เป็นช่องว่าง
> ซึ่งเหมาะกับงานเทียบข้อความ OCR แต่ทำให้ keyword ไทยพัง เพราะภาษาไทยไม่มีช่องว่างระหว่างคำ
>
> `"เรียก<ZWSP>เก็บคืนยา"` → แปลงเป็นช่องว่างได้ `"เรียก เก็บคืนยา"` แล้ว keyword `"เรียกเก็บคืนยา"` จับไม่ได้

---

## มารยาทต่อเว็บปลายทาง

`Fetcher` คุมให้อัตโนมัติ (คู่มือหน้า 3):

- ตรวจ `robots.txt` ก่อนทุก URL แล้ว cache ผลต่อ host
- หน่วง 1 วินาทีระหว่าง request ที่ไป host เดียวกัน เว็บต่างกันดึงพร้อมกันได้
- timeout 20 วินาที · `User-Agent: HerbRiskBot/1.0`
- 401/403/429 = `BlockedError` ไม่ใช่ error ธรรมดา

มีแฟล็ก `--no-robots` ไว้ตอนทดสอบเท่านั้น **ห้ามใช้ตอนรันจริง**

---

## เรื่อง HTTPS บนเครื่อง Windows

เครื่องที่มีแอนตี้ไวรัสดักตรวจ HTTPS (เช่น Avast) จะสวมใบรับรองของตัวเองเข้ามา
ซึ่งไม่อยู่ใน `certifi` ที่ Python ใช้โดยปริยาย ทำให้ขึ้น `CERTIFICATE_VERIFY_FAILED`
ทั้งที่เว็บปลายทางปกติดี

แก้ด้วย `truststore` (อยู่ใน `requirements.txt`) ให้ Python อ่าน certificate store ของ Windows
**ไม่ได้แก้ด้วย `verify=False`** เพราะนั่นคือการปิดการตรวจใบรับรองทิ้งทั้งหมด

---

## งานที่ยังไม่ได้ทำ (เป็นของฝั่ง backend)

- ตาราง `SearchKeyword`, `CrawlSource`, `KeywordSource`, `AnalysisLog`
- การนับ keyword
- ตั้งเวลา 03:00 (APScheduler) · API endpoint · UI

## แหล่งข่าวที่ยังค้าง

- **ต้องแก้ URL ก่อน 6 แหล่ง** — UNODC, INCB, ONDCP, Hong Kong MDD, สบส., Erowid
- **เข้าไม่ได้ 6 แหล่ง** — Reuters (`robots.txt` ห้าม), EMA News Search, DEA, EIN News, TGA ×2

รายละเอียดทั้งหมดอยู่ใน [SURVEY.md](SURVEY.md)
