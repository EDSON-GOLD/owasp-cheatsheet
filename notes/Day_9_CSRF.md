# Day 9 — CSRF (Cross-Site Request Forgery)

---

## สิ่งที่เรียนวันนี้

CSRF คืออะไร อธิบายสั้นๆ
Cross-Site Request Forgery การปลอมแปลง request ให้ดูเหมือนมาจาก user จริง โดยอาศัยว่า browser ส่ง cookie ไปให้ server อัตโนมัติ

ต่างจาก XSS กับ SQLi ยังไง
- **CSRF** มุ่งเน้นไปที่หลอกให้ server คิดว่า request มาจาก user จริง หรือหลอกให้ user ส่ง request โดยไม่รู้ตัว
- **XSS** มุ่งเน้นไปที่ ฉีด script รันใน browser
- **SQLi** มุ่งเน้นไปที่ ฉีด SQL รันใน Server

---

## CSRF มีกี่ประเภท อะไรบ้าง

- **Traditional CSRF** ใช้ form submission หลอกเหยื่อ
- **XMLHttpRequest CSRF** ใช้ JavaScript (AJAX/Fetch) ส่ง request แบบไม่ต้องโหลดหน้าใหม่ ไม่ต้องส่ง form ทั้งหน้า แค่ script รันเบื้องหลัง
- **Flash-based CSRF** ใช้ไฟล์ Flash (.swf) ส่ง request ปลอม (วิธีนี้ล้าสมัยแล้ว เนื่องจาก Adobe Flash ถูกยกเลิกปี 2020)

---

## วิธีป้องกัน CSRF

- **Anti-CSRF token** ที่ unpredictable
- **SameSite cookie** ตั้งเป็น Strict (แนะนำแบบ Strict จะครอบคลุมมากกว่าแบบ Lax) เนื่องจาก Lax ยังส่ง cookie กับ GET cross-site ได้ ส่วน Strict ส่งเฉพาะ request จาก domain เดียวกันเท่านั้น
- **Referrer Policy** จำกัดข้อมูลใน referer header
- **CSP** กำหนดแหล่ง content ที่เชื่อถือได้
- **Double-Submit Cookie** ที่ token ต้อง random จริงๆ
- **CAPTCHA** เป็น layer เสริม

ควรเลือกการป้องกันในหลากหลายรูปแบบควบคู่กัน (defence in depth) เพื่อให้ครอบคลุมช่องโหว่ในทุกมิติ

---

## สรุป 4 Challenge ที่ทำวันนี้

### Challenge 1: Hidden Link/Image

โจมตีโดยการสร้าง email หลอกให้เหยื่อกดลิงค์ ที่ซ่อนการโอนเงินไว้ด้านใน

ตัวอย่าง:
```html
<a href="http://mybank.thm:8080/dashboard.php?to_account=GB82MYBANK5698&amount=1000">
  Click Here to Redeem
</a>
```

ผลของการโจมตี คือ เมื่อคลิกลิงค์ browser ส่ง request ไปพร้อม session cookie ไปให้กับ server เมื่อ server ได้รับ cookie ถูกต้องจึงดำเนินการโอนเงินให้ attacker ทันที

### Challenge 2: Double Submit Cookie Bypass

ใน Challenge 2 มีระบบป้องกันอย่าง Double Submit Cookie ที่ทำการเช็ค cookie ของ browser และใน hidden field ของ form ตรงกันหรือไม่จึงจะเชื่อถือ

เป้าหมาย Challenge 2 คือควบคุมบัญชีของเหยื่อ

Attacker แบ่งโจมตีเป็น 3 ส่วน:

1. bypass CSRF token โดยเทคนิค Reverse engineer token ด้วยวิธีการนำ csrf-token ของตัวเองไปถอดรหัสใน CyberChef พบว่าระบบใช้การ encoding ด้วย base64 และใช้ข้อมูลเลขบัญชีเท่านั้น เมื่อทราบข้อมูลแล้ว Attacker ใช้การสร้าง csrf-token ปลอมของเป้าหมายโดยการ encoding ด้วย base64 โดยอิงจากเลขบัญชีของเหยื่อ
2. จากนั้นใช้เทคนิค subdomain inject cookie โดยคือการควบคุม subdomain เป้าหมายเพื่อ setup cookie ให้กับ domain หลักของธนาคาร เขียนทับ csrf-token cookie ของเหยื่อ
3. ส่งอีเมลหลอกเหยื่อ ให้ทำการกดลิงค์เพื่อเปลี่ยน Password เพื่อให้ข้อมูล Password ใหม่ถูกส่งไปที่ subdomain ของ Attacker แทน และ auto-submit form เปลี่ยน password ไปที่เว็บไซต์จริงของธนาคาร เมื่อ server ได้รับข้อมูล ทำการตรวจสอบ cookie กับ form token พบว่าตรงกันจึงดำเนินการเปลี่ยน password ใหม่ให้ทันที

### Challenge 3: SameSite Lax Logout

เป้าหมาย Challenge 3 คือออกจากระบบธนาคารของบัญชีเหยื่อ ด้วย Lax cookie

Attacker ส่ง email มี link ไป `logout.php` เป็นแบบ GET request Lax สามารถส่ง cookie ได้พร้อมกับ GET request ส่งผลให้สามารถออกจากระบบธนาคารของบัญชีเหยื่อได้

ปัญหาใน Challenge 3 คือ Lax สามารถส่ง cookie กับ GET cross-site ได้ ถ้าหากตั้งเป็น Strict จะป้องกันช่องโหว่นี้ได้ อีกทั้ง logout cookie ของระบบเป็นค่าคงที่ไม่เปลี่ยนแปลงทำให้สามารถใช้งานได้ตลอด

### Challenge 4: Lax+POST 2-minute window

เป้าหมาย Challenge 4 คือพัฒนาช่องโหว่ต่อเนื่องจาก Challenge 3 เพื่ออัพเดต logout cookie

เมื่อ Logout บัญชีเหยื่อได้แล้ว (cookie ถูก update) ภายใน 2 นาที ทำการส่ง POST เปลี่ยน isBanned = true โดยอาศัย Chrome exception ที่ cookie update ภายใน 2 นาที ส่งผลให้สามารถ POST logout cookie ได้

ปัญหาใน Challenge 4 คือการต่อยอดจากปัญหาใน Challenge 3 โดยเพิ่มจากตอนแรกแค่ทำการ GET + logout cookie โดยใช้ Lax แต่ใน Challenge นี้ ทำการ GET + logout cookie โดยใช้ Lax + POST 2-minute เพื่อ update cookie

---

## จุดที่เข้าใจผิดและต้องแก้ไข

### 1. CSRF ย่อมาจากอะไร — ตอบผิดใน Pretest

**เข้าใจผิด:** คิดว่า CSRF ย่อมาจาก Client-Side Request Forwarding

**ที่ถูกต้อง:** CSRF = Cross-Site Request **Forgery** — Forgery แปลว่า "การปลอมแปลง" ซึ่งตรงกับสิ่งที่ช่องโหว่นี้ทำ คือปลอมแปลง request ให้ดูเหมือนมาจาก user จริง

### 2. base64 คิดว่าเป็น hash — สับสนระหว่างเขียนโน้ต

**เข้าใจผิด:** เรียก base64 ว่า "hash ด้วย base64"

**ที่ถูกต้อง:** base64 เป็น **encoding** ไม่ใช่ hash — encoding แปลงกลับได้เสมอ แต่ hash เป็นทางเดียวกลับคืนไม่ได้ นี่คือเหตุผลที่ attacker ถอดรหัส CSRF token ได้ง่าย

### 3. CAPTCHA อย่างเดียวป้องกัน CSRF ได้ — สงสัยระหว่างทำ Posttest

**เข้าใจผิด:** คิดว่า CAPTCHA น่าจะป้องกัน CSRF ได้เพราะหลายเว็บใช้

**ที่ถูกต้อง:** CAPTCHA ป้องกันแค่บางจุด ไม่ได้ใส่ทุก request และ CSRF ไม่ใช่ bot แต่เป็น browser ของ user จริงส่ง request เอง ต้องป้องกันหลายชั้น (defence in depth)

---

## Lesson สำคัญที่สุดของวันนี้

- **การคลิก link ในอีเมลที่ไม่ปลอดภัย 1 ครั้งสามารถสร้างผลกระทบได้มากกว่าที่คิด** — วันนี้เป็นการเห็นการทำงานจริงของ Attacker ในการใช้งานช่องโหว่จากระบบ และได้เข้าใจการทำงานของ request มากขึ้นในการกดคลิก 1 ครั้งเกิดอะไรขึ้นบ้าง เช่น GET/POST request, cookie เป็นต้น อีกทั้งยังได้รู้ว่าการส่ง cookie ไปกับ cross-site request ตั้งแค่ระดับกลางอย่าง Lax นั้นไม่ได้ปลอดภัยอย่างที่คิด และการได้เรียนรู้ XSS มาก่อนทำให้เห็นภาพมากยิ่งขึ้นจากในตอนแรกแค่ใส่ JavaScript สั้น แต่ CSRF มีเพิ่มเติมเรื่อง cookie เสริมมาด้วย

- **CSRF token จำเป็นต้อง unpredictable จริงๆ** ไม่ใช่แค่ใส่ token เฉยๆ ไม่เพียงพอ เพราะถ้าหาก token เดาง่าย เช่น base64 ของเลขบัญชี ตาม Challenge 2 นั้น attacker สามารถ reverse engineer ได้ทันที ใน Day 1-7 จะเห็นว่า parameterized query ต้องทำทุกจุดจึงจะป้องกันได้ครอบคลุม CSRF token ก็เช่นกันต้องออกแบบให้ดีทุกจุด

- **defence in depth นั้นสามารถใช้ได้กับทุกช่องโหว่** ในตัวอย่าง Day 8 XSS จะเห็นว่าต้องทำการป้องกันหลากหลายรูปแบบ sanitize + escape + HttpOnly จึงจะปลอดภัย และใน CSRF ก็เหมือนกันจำเป็นต้อง CSRF token + SameSite + CSP + Referrer Policy ป้องกันชั้นเดียวนั้นไม่เพียงพอ

---

*Day 9 Complete — ต่อไป: Day 10*
