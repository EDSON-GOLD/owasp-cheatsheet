# Day 1 — SQL Injection Part 1

## สิ่งที่เรียนวันนี้

Introduction to SQL Injection: Part 1 การแทรก Payload ลงในจุดที่เป็นช่องโหว่ของระบบ ด้วยคำสั่ง SQL โดยที่ใน SQL Injection: Part 1 อธิบายถึงรูปแบบการ Injection ทั้งหมด 4 แบบได้แก่

1. SQL Injection 1: Input Box Non-String
2. SQL Injection 2: Input Box String
3. SQL Injection 3: URL Injection
4. SQL Injection 4: POST Injection

ผลที่เกิดขึ้นจากการ inject คือการสามารถ Bypass Log-in เข้าสู่ระบบได้โดยไม่ต้องใช้ข้อมูลใดๆ ส่งผลให้สามารถ Disclosure ซึ่งเป็นหนึ่งใน The DAD Triad Security

---

## SQL Injection คืออะไร (อธิบายด้วยคำตัวเอง)

ขอยกตัวอย่าง การที่โจรสามารถเข้าบ้านคุณได้ โดยที่โจรไม่จำเป็นต้องรู้ด้วยซ้ำว่าคุณใช้กุญแจลักษณะไหน ยี่ห้ออะไร ทำงานยังไง ซึ่งโจรใช้วิธีทั่วไปอย่างการสะเดาะกุญแจ

- **กุญแจ** คือ ภาษา SQL
- **การสะเดาะกุญแจ** คือ การใช้ประโยชน์จากหลักการทำงานของ SQL
- **บ้านคุณ** คือ Database ที่ใช้เก็บทรัพย์สินของคุณ

---

## Payload ที่ทดลองทำครั้งแรก แล้วไม่ได้ผล(ส่วนตัว)
OR 1=1
OR 1=1 --
เข้าใจว่าสามารถใช้แบบนี้ได้กับทุก Challenge เพราะเคยลองเล่น OWASP Juice Shop มาบ้าง



### Challenge 1: Integer field (SQL Injection 1: Input Box Non-String)

- **Payload:** `1 or 1=1-- -`
- **Input Type:** Integer — ระบบรับค่าแบบตรงๆ ไม่ต้องมี `'` นำหน้าก็สามารถ Injection ได้

**อธิบายทีละส่วน:**

| ส่วน | ความหมาย |
|------|-----------|
| `1` | ค่าที่ใส่เข้าไปใน field |
| `or 1=1` | เงื่อนไขที่เป็นจริงเสมอทุก row (ทางคณิตศาสตร์ เหมือน true ที่ตายตัว) |
| `-- -` | Comment ใน SQL — ทุกอย่างหลังเครื่องหมายนี้จะถูก Database มองข้ามทั้งหมด |

**สรุป:** ขอผ่านเข้าบ้านของคุณ โดยที่ไม่สนใจอะไรทั้งนั้น

---

### Challenge 2: String field (SQL Injection 2: Input Box String)

- **Payload:** `1' or '1'='1'-- -`
- **Input Type:** String — ระบบรับค่าแบบมีเงื่อนไข ต้องมีอักษรพิเศษ `'` ต่อท้ายจึงจะสามารถ Injection ได้

**อธิบายทีละส่วน:**

| ส่วน | ความหมาย |
|------|-----------|
| `1'` | ปิด string เดิมก่อน แล้วค่อยต่อ payload |
| `or '1'='1'` | เงื่อนไขที่เป็นจริงเสมอทุก row เหมือน `1=1` แต่อยู่ในรูปแบบ String |
| `-- -` | Comment ตัดทุกอย่างหลังจากนั้นทิ้ง |

**เหตุผลที่ใช้ `'1'='1'` แทน `1=1`:**
บาง WAF หรือ filter block integer condition อย่าง `1=1` แต่ไม่ block string condition อย่าง `'1'='1'` — เป็นการหลีกเลี่ยง filter บางประเภทที่ระบบมีการตรวจสอบอยู่

---

### Challenge 3: URL Injection (SQL Injection 3: URL Injection)

- **Payload:** `login?profileID=-1' or 1=1-- -&password=1`
- **Input Type:** URL — ช่องโหว่นี้เกิดขึ้นกับตำแหน่ง URL ของระบบ สามารถนำ Payload ไปใส่ตรงๆ ได้เลย

อาจใช้เทคนิค Integer field หรือ String field ขึ้นอยู่กับระบบ หลักการเดียวกับ Challenge 1 และ 2 แต่เปลี่ยนจุดที่ใส่ Payload มาอยู่ที่ URL แทน

---

### Challenge 4: POST Injection (SQL Injection 4: POST Injection)

- **วิธี:** ใช้ Burp Suite intercept request แล้วแก้ไข Body ก่อน Forward

Challenge นี้แตกต่างจาก 3 Challenge ก่อนหน้า เพราะจำเป็นต้องรู้จักหลักการทำงานของการส่งข้อมูล

**เปรียบเทียบกับการส่งจดหมาย:**
- **GET** → ถามว่ามีอะไรอยู่ในบ้านไหม — ข้อมูลอยู่ใน URL → ใส่ Payload ใน URL ได้เลย
- **POST** → ส่งพัสดุเข้าบ้าน — ข้อมูลอยู่ใน body → ต้องแก้ไขข้อมูลระหว่างทางก่อนที่จะส่งถึงปลายทาง

**ทำไมต้องใช้ Burp Proxy + Forward ไม่ใช่ Repeater:**
Repeater ส่ง session cookie เก่าไป server มองว่ายังไม่ได้ login — ต้องใช้ Proxy + Forward ผ่าน Burp Browser เพราะ browser จัดการ session ให้อัตโนมัติ

---

## จุดที่เข้าใจผิดตอนแรก

- **`1=1` ไม่ใช่ user ID ที่ 1** — มันคือเงื่อนไขทางคณิตศาสตร์ที่เป็นจริงเสมอทุก row เหมือน true ที่ตายตัว ทำให้ WHERE condition เป็น true ทั้งตาราง

- **`-1` ใช้เพื่อ** กำหนดให้ Input รับค่าที่ไม่มีอยู่จริงในระบบ เมื่อกำหนดในรูปแบบนี้ Database จะไปอ่านเงื่อนไขถัดไปคือ `or 1=1-- -` แทน ส่งผลให้คืน row แรกในตาราง (มักเป็น admin) มาให้

- **GET vs POST** เข้าใจผิดว่า GET = นำเข้า / POST = ส่งออก
  - **GET** → ขอดูข้อมูลจาก server → ข้อมูลอยู่ใน URL
  - **POST** → ส่งข้อมูลให้ server ประมวลผล → ข้อมูลอยู่ใน body

---

## สิ่งที่ได้เรียนรู้เพิ่มเติม

ในครั้งแรกที่ทบทวน เข้าใจมาตลอดว่า `or 1=1` สามารถใช้ได้กับทุกๆ ช่องโหว่ที่เกี่ยวกับการ Log-in แต่หลังจากได้ทบทวนและฝึก Challenge ทำให้ทราบถึง

- Input Type มีอะไรบ้าง และส่งผลต่อรูปแบบ Payload ยังไง
- ลักษณะการใช้งาน `or 1=1` ที่เกี่ยวข้องกับ Input Type
- เทคนิคการใช้ `-1` เพื่อข้ามเงื่อนไข
- การอธิบาย `--` ว่ามีไว้ทำไมใน SQL

---

*Day 1 Complete — ต่อไป: Introduction to SQL Injection Part 2*