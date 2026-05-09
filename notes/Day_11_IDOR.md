# Day 11 — IDOR (Insecure Direct Object Reference)

---

## สิ่งที่เรียนวันนี้

IDOR (Insecure Direct Object Reference) คือช่องโหว่ที่เกิดจากการที่ Server side ไม่ได้มีการตรวจสอบสิทธิ์ก่อนให้เข้าถึงข้อมูลทรัพยากรในระบบ

IDOR อยู่ภายใต้ **A01: Broken Access Control** ใน OWASP Top 10

### IDOR มี 2 แบบหลักๆ

1. **แบบที่สามารถเข้าถึงข้อมูลของคนอื่น** — เช่น ดู profile, invoice, order ของผู้ใช้งานอื่น
2. **แบบที่สามารถทำ Action แทนคนอื่น** — เช่น ลบ account, เปลี่ยน password ของผู้ใช้งานอื่น

### ID ถูกซ่อนได้หลายรูปแบบ

- **Encoded IDs** — เช่น Base64 encode ค่า ID แต่ attacker สามารถ decode → แก้ค่า → encode กลับ → ส่งใหม่ได้
- **Hashed IDs** — เช่น MD5 hash ค่า ID แต่ถ้าค่าเดิมเป็นตัวเลขง่ายๆ สามารถใช้ lookup table อย่าง crackstation.net จับคู่ hash กับค่าเดิมได้
- **Unpredictable IDs** — เช่น UUID ที่เดาไม่ได้ ต้องใช้วิธีสร้าง 2 accounts แล้วสลับ ID กันเพื่อทดสอบ

**หลักการสำคัญ:** encoding ≠ encryption ≠ access control — ทั้งหมดเป็นการปกปิด raw data เท่านั้น ไม่ใช่ root fix ของ IDOR

### IDOR ซ่อนอยู่ได้หลายที่

ไม่จำเป็นต้องอยู่ใน URL เสมอไป สามารถซ่อนได้ทั้งใน POST body, cookie, HTTP header, API request, AJAX request ที่ browser โหลดในพื้นหลัง และ unreferenced parameter ที่ developer ใช้ตอน dev แล้วลืมปิดก่อน push production

---

## TryHackMe Room: IDOR

### Challenge 1: IDOR Example

เป็น Challenge ที่ต้องทดลองทำการเปลี่ยนข้อมูล ID Order คำสั่งซื้อ เพื่อให้สามารถเข้าถึงข้อมูลคำสั่งซื้ออื่นๆ ที่ไม่ใช่คำสั่งซื้อของตนเองได้

โดยเป็นการแก้ไข URL จากเดิมคือ `https://onlinestore.thm/order/1234/invoice` เปลี่ยนเป็น `https://onlinestore.thm/order/1000/invoice`

ผลลัพธ์: เห็น invoice ของผู้ใช้งานอื่นทันที

### Challenge 2: A Practical IDOR Example

เป็น Challenge ที่อธิบายหลักการคล้ายกันกับ Challenge ที่ 1 แต่จะใช้วิธีที่ซับซ้อนกว่าคือการอ่านค่า request ในหน้า Network tab ของ Developer Tools เพื่อหา request ที่ทำหน้าที่ GET ข้อมูลของ User account มาแสดง เพื่อทำการแก้ไขข้อมูล ID เป็น ID อื่นๆ ที่มีในระบบ

ผลลัพธ์: เข้าถึงข้อมูล Username, Email ของผู้ใช้งานอื่นในระบบ

API request endpoint: `/api/v1/customer?id={user_id}`

### ความต่างที่สำคัญระหว่าง Challenge 1 กับ 2

- **Challenge 1** — ไม่ได้มีการปกปิดข้อมูล ID ของ Order ระบบแสดงข้อมูล ID ให้เห็นที่ URL ตรงๆ ซึ่งสามารถเปลี่ยนแปลงข้อมูลได้เลย
- **Challenge 2** — ระบบถูกออกแบบมาให้ดียิ่งขึ้น มีการปกปิดข้อมูล ID ของ User บน URL และใช้การเรียกใช้งาน request GET ข้อมูล Account ในพื้นหลังทดแทน ซึ่งการที่จะทราบว่ามี request พื้นหลังหรือไม่ จำเป็นต้องเปิด Developer Tools ใช้ Network tab เพื่อตรวจสอบ request ทั้งหมดเพื่อหา endpoint ที่ต้องการ

**บทเรียน:** IDOR ไม่ได้ซ่อนแค่ใน address bar — ต้องเช็ค request เบื้องหลังด้วยเสมอ

---

## จุดที่เข้าใจผิดและต้องแก้ไข

### 1. ไม่รู้ชื่อเต็ม IDOR

**เข้าใจผิด:** ไม่สามารถตอบได้ว่า IDOR ย่อมาจากอะไร เพราะปกติไม่เคยรู้ว่าช่องโหว่แบบนี้จัดอยู่ในหมวดหมู่อะไร แต่เคยเจอปัญหาประเภทนี้มาบ้าง

**ที่ถูกต้อง:** IDOR = Insecure Direct Object Reference อยู่ภายใต้ A01: Broken Access Control ใน OWASP Top 10

### 2. ตอบวิธีป้องกัน IDOR กว้างเกินไป

**เข้าใจผิด:** ตอบแค่ "กำหนด permission ของข้อมูล และสิทธิ์การเข้าถึงข้อมูล" ซึ่งไม่เฉพาะเจาะจง

**ที่ถูกต้อง:** Root fix ของ IDOR คือ (1) Server-side authorization check — ทุกๆ request ที่เกิดขึ้นจาก client side, server side ต้องเช็คว่า user นั้นๆ เป็นเจ้าของ resource นั้นจริงๆ (2) ใช้ indirect reference แทนที่จะใช้ `id=number` ตรงๆ ให้เปลี่ยนไปใช้ค่าที่ map กับ session ของ user นั้นๆ ทดแทน เช่น UUID ที่คาดเดาไม่ได้

### 3. สับสนประเภทของ IDOR กับวิธีซ่อน ID

**เข้าใจผิด:** ตอบว่า IDOR แบ่งออกเป็น Encoded IDs, Hashed IDs, Unpredictable IDs

**ที่ถูกต้อง:** นั่นคือ "วิธีที่ ID ถูกซ่อน" ไม่ใช่ "ประเภทของ IDOR" — ประเภทจริงๆ คือ (1) แบบที่สามารถเข้าถึงข้อมูลของคนอื่น (2) แบบที่สามารถทำ Action แทนคนอื่น

### 4. แยก Logic Flaw กับ Client-side Bypass ไม่ออก (จาก Review Test Day 10)

**เข้าใจผิด:** ยกตัวอย่างหน้า reset password ที่ใช้ `$_REQUEST` ว่าเป็น client-side bypass

**ที่ถูกต้อง:** กรณี `$_REQUEST` เป็น server-side logic flaw — เป็นปัญหาที่ server ประมวลผลผิด ไม่ใช่ client-side validation ที่ถูก bypass ตัวอย่าง client-side bypass จริงๆ คือ JavaScript validation ที่บล็อก input แต่ใช้ Burp Suite intercept request แล้วแก้ payload ส่งตรงไป server ได้

---

## Finding

### Finding 1: IDOR Order Confirmed (Challenge 1)

- **Severity:** High
- **Root cause:** Server ไม่ตรวจเช็คสิทธิ์ของผู้ใช้งาน ก่อนให้เข้าถึง resource ในระบบ
- **Impact:** Attacker สามารถแก้ไขข้อมูล URL parameter ของคำสั่งซื้อของตนเอง เปลี่ยนเป็นหมายเลขออเดอร์ของผู้ใช้งานอื่นได้ เพื่อเข้าถึงข้อมูลคำสั่งซื้อ
  - URL: `https://onlinestore.thm/order/1234/invoice`
- **Remediation:** ดำเนินการตั้งค่า Server-side authorization check ทุก request ต้องเช็คว่า user นี้เป็นเจ้าของ resource ข้อมูลหมายเลขออเดอร์ที่ร้องขอจริงไหม และควรซ่อน ID ของ Order ไม่ให้แสดงใน URL parameter พร้อมใช้ indirect reference แทนการใช้ข้อมูลตรงๆ เลือกใช้ค่าการ map session ของ user account นั้นๆ ทดแทน

### Finding 2: IDOR Your Account Tab (Challenge 2)

- **Severity:** Critical
- **Root cause:** Server ไม่ตรวจเช็คสิทธิ์ของผู้ใช้งาน ก่อนให้เข้าถึง resource ในระบบ
- **Impact:** Attacker สามารถแก้ไขข้อมูล customer ID ผ่าน URL และสามารถเข้าถึงข้อมูล username, email ของผู้ใช้งานอื่นได้ทันที โดยใช้วิธีตรวจสอบ request พื้นหลังที่ถูกซ่อนผ่านการเปิด Developer Tools ใช้ Network tab
  - API request endpoint: `/api/v1/customer?id={user_id}`
- **Remediation:** ดำเนินการตั้งค่า Server-side authorization check ทุก request ต้องเช็คว่า user นี้เป็นเจ้าของ resource ข้อมูล Your Account ที่ร้องขอจริงไหม และควรซ่อน ID ของ customer ไม่ให้แสดงใน API request พร้อมใช้ indirect reference แทนการใช้ข้อมูลตรงๆ เลือกใช้ค่าการ map session ของ user account นั้นๆ ทดแทน
- **Note — เหตุผลที่จัด Severity: Critical:** ช่องโหว่ดังกล่าวสามารถสร้าง impact chain ส่งผลให้เกิดการ account takeover ได้ผ่านเมนู reset password เพราะเมนูดังกล่าวเช็คข้อมูล username, email ของผู้ใช้งานถูกต้องหรือไม่ หากถูกต้องจะสามารถ reset password ได้ทันที
- **Note — เหตุผลที่ Finding 1 High แต่ Finding 2 Critical:** Finding 1 สามารถเข้าถึงข้อมูลของผู้อื่นได้ แต่ครอบคลุมแค่ในส่วนของ Order Confirmed เท่านั้น Finding 2 ให้ Critical เพราะสามารถต่อยอดความเสียหายได้มากกว่าการเข้าถึงข้อมูลผู้ใช้งานคนอื่น กระทบ impact chain ต่อเนื่องโดยตรง

---

## Lesson สำคัญที่สุดของวันนี้

- **IDOR เป็นอีกเรื่องที่เคยมีประสบการณ์ในตอนทำงานมาบ้าง** — เคยพบช่องโหว่ดังกล่าวค่อนข้างบ่อย เพราะมีความเกี่ยวข้องกับเรื่องของ Permission Level ของ OS ที่ระบบนั้นๆ ทำงานอยู่ แล้วไม่เคยทราบเลยว่าช่องโหว่แบบนี้เรียกว่าอะไร จัดประเภทยังไง ครอบคลุมยังไง พอได้เรียนรู้ก็ทราบว่าช่องโหว่ประเภทนี้อยู่ภายใต้ A01: Broken Access Control ใน OWASP Top 10 ซึ่งเป็นชนิดของช่องโหว่ที่พบได้บ่อยที่สุด

- **IDOR ไม่ได้มีแค่การเข้าถึงข้อมูลของคนอื่นเท่านั้น** — ยังครอบคลุมไปถึงการทำ action แทนคนอื่นอีกด้วย โดย root cause คือ server ไม่เช็คสิทธิ์ก่อนให้เข้าถึง resource นั้นๆ ส่งผลให้เกิดช่องโหว่

- **Encoding/Hashing ไม่สามารถป้องกัน IDOR ได้** — หลักการ encoding ≠ encryption ≠ access control ทั้งหมดนั้นเป็นการปกปิด raw data เท่านั้น ไม่ใช่ root fix ของ IDOR แม้ hash จะไม่สามารถถอดรหัสได้ แต่ถ้า hash เป็นค่าที่ง่ายเกินไป เช่น ตัวเลขจำนวนเต็ม 3 ตัวเลข การใช้เทคนิค match hash กับฐานข้อมูล hash อย่าง crackstation.net เพื่อตรวจสอบความเข้ากันได้ ถ้าถูกต้องก็สามารถเดาได้ว่า hash นั้น raw data คืออะไร

- **ประสบการณ์จริงเชื่อมกับ IDOR ได้โดยตรง** — ระบบถูกกำหนด Permission Level 777 ทั้ง path เก็บไฟล์ PDF และให้สิทธิ์ www-data สามารถเข้าถึงได้ทั้งหมด ผลก็คือทุกคนที่เปิดหน้าเว็บสามารถเข้าถึงไฟล์ PDF ได้ทุกไฟล์ อีกทั้ง attacker สามารถแนบไฟล์ PDF สอดไส้การ redirect ไปยังเว็บพนันได้ เพราะระบบให้สิทธิ์ execute — วิธีแก้ไขในตอนนั้นคือการ change Permission Level จาก 777 → 644 ในแต่ละไฟล์ หลังจากได้เรียน IDOR จึงได้รู้ว่าการปรับ Permission Level ควรทำควบคู่กับการที่ developer ตั้งค่า Server-side authorization check ทุกๆ request

- **Attacker คิดแบบ business model** — ไม่ใช่แค่ทำลาย การลบไฟล์เป็นการบอกว่าสามารถทำอะไรได้บ้าง แต่การทำ redirect ผ่าน token ของ attacker โดยตรงจะสามารถได้เงินจากเว็บไซต์พนันง่ายกว่า

---

*Day 11 Complete — ต่อไป: Day 12 OWASP ที่เหลือ*
