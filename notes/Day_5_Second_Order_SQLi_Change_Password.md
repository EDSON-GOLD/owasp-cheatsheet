# Day 5 — Second-Order SQL Injection: Change Password

---

## สิ่งที่เรียนวันนี้

Day 5 ค่อนข้างไม่ยากมาก จากระดับของ Challenge จัดอยู่ในระดับ Medium ซึ่งต่างจาก Day 4 ที่จัดอยู่ในระดับ Hard และมีขั้นตอนที่ง่ายกว่า ภาพรวมของช่องโหว่มีลักษณะที่คล้ายกัน แต่แตกต่างกันในมุมมองของผลกระทบที่มีต่อระบบ — Day 4 คือ **data breach**, Day 5 คือ **account takeover** โดยทั้ง 2 Challenge นั้นมี root cause ที่เหมือนกันคือเลือกใช้งาน string concatenation แทนการใช้งาน parameterized query ส่งผลให้เกิดช่องโหว่ขึ้น

จุดสำคัญของ Day 5 ที่แตกต่างจาก Day 4 คือความร้ายแรงในมุมของ real-world scenario — Day 4 นั้นสามารถ dump ข้อมูลจากระบบได้จริง ความเสียหายที่เกิดขึ้นเป็นวงกว้างและกระทบผู้คนหลายคนจริง แต่ยังอยู่ในขอบเขตของการอ่านข้อมูลเท่านั้น แต่ใน Day 5 การเข้าถึง admin account จริงๆ ของระบบนั้นได้ ส่งผลในหลากหลายมิติมากกว่า เช่น ควบคุมระบบทั้งหมดได้ ลบข้อมูล, เพิ่ม user, เปลี่ยน config, เข้าถึงข้อมูลทุก user หรือแม้แต่ใช้ระบบนั้นเป็นจุดเริ่มต้นในการโจมตีระบบอื่นต่อไป (lateral movement / pivot)

---

## Second-Order SQL Injection บน Change Password คืออะไร

Challenge นี้จัดอยู่ในหมวดหมู่ **Second-Order SQL Injection** โดยทำงานผ่านฟังก์ชัน Change Password

Developer ได้แก้ไขช่องโหว่ที่หน้า login และ notes แล้ว แต่เพิ่มฟังก์ชัน Change Password ใหม่เข้ามา ซึ่งมีช่องโหว่เกิดขึ้นเพราะ developer มองว่า username ไม่ได้มาจาก user โดยตรง แต่คือการดึงข้อมูลจาก database ผ่าน session id จึงคิดว่าเชื่อถือได้และไม่ต้องใช้ placeholder

นี่คือความผิดพลาดในการคิด — **การแบ่ง input เป็น "เชื่อถือได้" กับ "เชื่อถือไม่ได้" ตามแหล่งที่มา** แทนที่จะใช้ parameterized query กับทุกค่าที่เข้า query ส่งผลให้เกิดช่องโหว่ Second-Order SQL Injection ได้

**Query ที่มีช่องโหว่:**
```sql
UPDATE users SET password = ? WHERE username = '" + username + "'
```

- `password` → ใช้ `?` placeholder เพราะ developer รู้ว่า user พิมพ์เข้ามาเอง = อันตราย
- `username` → ใช้ string concatenation เพราะ developer คิดว่ามาจาก DB ของเราเอง = ปลอดภัย

### Flow การโจมตี

1. **Register** user ชื่อ `admin'-- -` → ระบบเก็บชื่อนี้ลง database ตามปกติ ยังไม่มีอะไรเกิดขึ้น
2. **Login** เป็น `admin'-- -` → ได้ session id ของ user นี้
3. **ไปหน้า Change Password** → ใส่ password ใหม่
4. **ระบบทำการเปลี่ยน Password** → ดึง username จาก DB มาต่อ string → payload ถูก execute
5. **Password ใหม่ถูกเขียนทับ** ที่ user `admin` ตัวจริง

เปรียบเทียบแบบเข้าใจง่าย — เหมือนกล่องพัสดุที่เขียนชื่อ `admin'-- -` ตอนเก็บเข้าคลังเก็บตามชื่อเต็มไม่มีปัญหา แต่ตอนระบบอ่านชื่อออกมาเพื่อส่งกุญแจใหม่ อ่านได้แค่ `admin` แล้วตัดส่วนที่เหลือทิ้ง เลยส่งกุญแจไปให้ admin ตัวจริงแทน

---

## ขั้นตอนการ Exploit

### Step 1 — Register user ด้วย payload เป็น username

```sql
-- ระบบตรวจสอบว่า username ซ้ำหรือไม่
SELECT username FROM users WHERE username=?

-- เก็บ user ใหม่ลง database (parameterized query = payload ถูกเก็บเป็น data)
INSERT INTO users (username, password) VALUES (?, ?)
```

> Username ที่ register: `admin'-- -`
> Parameterized query เก็บ payload ทั้งก้อนเป็น plain text — ยังไม่มีอะไรเกิดขึ้น

### Step 2 — Login ด้วย payload ทั้งก้อนเป็น username

```sql
SELECT id, username FROM users WHERE username = ? AND password = ?
```

> ต้องพิมพ์ `admin'-- -` ทั้งหมดเป็น username เพราะ parameterized query เก็บ payload เป็นชื่อ user

### Step 3 — เปิดหน้า Change Password และใส่ password ใหม่

### Step 4 — ระบบ execute query ที่มีช่องโหว่

```sql
-- ระบบดึง username จาก session
SELECT username, password FROM users WHERE id = ?

-- ระบบนำ username มาต่อ string ตรงๆ → payload ทำงาน
UPDATE users SET password = ? WHERE username = 'admin'-- -'
```

**`-- -` คือ SQL comment** — ตัดทุกอย่างหลังมันทิ้ง รวมถึง `'` ตัวสุดท้ายที่เป็นของ query เดิม

ระบบจึงเปลี่ยน password ของ `admin` ตัวจริง ไม่ใช่ `admin'-- -`

### Step 5 — Login เป็น admin ด้วย password ใหม่

```sql
SELECT id, username FROM users WHERE username = ? AND password = ?
-- Parameters: admin, <password_ใหม่ที่ตั้งไว้>
```

Login สำเร็จ → เข้าถึง admin account → ได้ flag

---

## จุดที่เข้าใจผิดและต้องแก้ไข

### 1. ทำไม developer คิดว่าปลอดภัย — เข้าใจผิดในตอนแรก

**เข้าใจผิด:** คิดว่า developer มั่นใจเพราะแก้ไขฟังก์ชัน login แล้ว

**ที่ถูกต้อง:** Developer แบ่ง input เป็น "เชื่อถือได้/ไม่ได้" ตามแหล่งที่มา — password มาจาก user โดยตรงจึงใช้ `?` ป้องกัน แต่ username มาจาก database ผ่าน session id จึงคิดว่าปลอดภัยและไม่ต้องป้องกัน

### 2. `-- -` ทำหน้าที่อะไร — เข้าใจผิดในตอนแรก

**เข้าใจผิด:** คิดว่าเกี่ยวกับ placeholder หรือ parameterized query ที่ทำให้ส่วนที่เหลือไม่ถูกประมวลผล

**ที่ถูกต้อง:** `-- -` คือ SQL comment ที่ตัดทุกอย่างหลังมันทิ้ง — เรื่องเดียวกับที่เรียนใน Day 1 ที่ใช้ `admin'-- -` ตัด `AND password = '...'` ออก

### 3. String concatenation กับ parameterized query — สลับกันในตอนแรก

**เข้าใจผิด:** บอกว่า payload ถูกเก็บเป็น data ผ่าน string concatenation

**ที่ถูกต้อง:**
- **Parameterized query** (`?`) → ปลอดภัย → มอง input เป็น data เสมอ
- **String concatenation** (`'" + username + "'`) → อันตราย → input กลายเป็น code ได้

### 4. Analogy กล่องพัสดุ — เข้าใจผิดในตอนแรก

**เข้าใจผิด:** คิดว่าปัญหาคือ "กล่องเหมือนกัน" จึงส่งของไปให้ผิดคน

**ที่ถูกต้อง:** ปัญหาคือ **ชื่อบนกล่องถูกอ่านผิด** — ตอนเก็บเข้าคลังเก็บชื่อเต็ม `admin'-- -` ไม่มีปัญหา แต่ตอนระบบอ่านชื่อออกมาใช้งาน อ่านได้แค่ `admin` แล้วตัดส่วนที่เหลือทิ้ง

---

## Finding ที่ควรแจ้งลูกค้า

### Finding 1: Second-Order SQL Injection ที่หน้า Change Password
- **Severity:** Critical
- **Root cause:** Query เปลี่ยน password ใช้ string concatenation กับ username ไม่ใช้ parameterized query
- **Impact:** Attacker สามารถเข้าถึง admin account ซึ่งมีสิทธิ์สูงสุดภายในระบบ — ลบข้อมูล, เพิ่ม user, เปลี่ยน config, เข้าถึงข้อมูลทุก user และอาจใช้เป็นจุดเริ่มต้นโจมตีระบบอื่นต่อ (lateral movement)
- **Remediation:** เปลี่ยนเป็น `WHERE username = ?` เหมือน password ที่ทำไว้แล้ว

### Finding 2: Parameterized query ไม่ครบทุก query
- **Severity:** High
- **Root cause:** Developer แบ่ง input เป็น "เชื่อถือได้/ไม่ได้" ตามแหล่งที่มา แทนที่จะใช้ parameterized query กับทุก query
- **Impact:** แม้จุดรับข้อมูลจะปลอดภัย แต่จุดที่ดึงข้อมูลออกมาใช้ยังมีช่องโหว่
- **Remediation:** ทุก query ที่มี user input หรือข้อมูลจาก database ต้องใช้ parameterized query ไม่ใช่แค่บางจุด

---

## เปรียบเทียบ Day 4 vs Day 5

| | **Day 4** | **Day 5** |
|---|---|---|
| **ประเภท** | Second-Order SQL Injection | Second-Order SQL Injection |
| **จุดที่ฉีด** | Register username | Register username |
| **Function ที่ trigger** | เปิดหน้า Notes | Change Password |
| **Query ที่มีช่องโหว่** | SELECT (อ่านข้อมูล) | UPDATE (แก้ไขข้อมูล) |
| **Impact** | Data breach — dump ข้อมูลทั้ง DB | Account takeover — ยึด admin account |
| **Root cause** | String concatenation ใน SELECT | String concatenation ใน UPDATE |
| **Remediation** | เปลี่ยนเป็น `WHERE username = ?` | เปลี่ยนเป็น `WHERE username = ?` |

---

## Lesson สำคัญที่สุดของวันนี้

- **Parameterized query (`?`) → ปลอดภัย → มอง input เป็น data เสมอ** ส่วน **String concatenation (`'" + username + "'`) → อันตราย → input กลายเป็น code ได้** — นี่คือหลักการพื้นฐานที่ต้องจำให้ขึ้นใจ

- **อย่าแบ่ง input เป็น "เชื่อถือได้/ไม่ได้" ตามแหล่งที่มา** — ข้อมูลที่มาจาก database ก็อันตรายได้เหมือนกัน ถ้ามันเคยเป็น user input มาก่อน ต้องใช้ parameterized query กับทุกค่าที่เข้า query โดยไม่มีข้อยกเว้น

- **Account takeover อาจร้ายแรงกว่า data breach** — การเข้าถึงข้อมูลได้แค่ 1 บัญชีอาจฟังดูน้อย แต่หากบัญชีนั้นคือบัญชีที่สำคัญ กระทบอีกหลายล้านบัญชี นั่นคือปัญหาอีกระดับ admin สามารถควบคัมระบบทั้งหมด ลบข้อมูล เปลี่ยน config หรือใช้เป็นจุดเริ่มต้นโจมตีระบบอื่น (lateral movement / pivot)

- **คำศัพท์ใหม่ที่ได้วันนี้:**
  - **Data breach** — เหตุการณ์ที่ข้อมูลสำคัญถูกเข้าถึง, ขโมย, เปิดเผย หรือใช้งานโดยผู้ที่ไม่ได้รับอนุญาต
  - **Account takeover** — การเข้ายึดบัญชีโดยผู้ที่ไม่ได้รับอนุญาต
  - **Lateral movement / pivot** — ใช้ระบบที่ยึดได้เป็นจุดเริ่มต้นโจมตีระบบอื่นต่อ

---

*Day 5 Complete — ต่อไป: Day 6*
