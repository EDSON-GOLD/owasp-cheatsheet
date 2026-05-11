# OWASP Quick Reference — Day 12

*OWASP Top 10 ที่เหลือ — สรุปสั้นเพื่ออธิบายได้ใน interview*

---

## 1. SSRF (Server-Side Request Forgery)

SSRF คือการหลอกให้ server ส่ง request ไปยัง resource ที่ attacker กำหนด

- **Severity:** Critical
- **Root cause:** server ไม่ได้มีการ validate/จำกัด URL ปลายทางก่อน fetch
- **Impact:** attacker ยืมสิทธิ์จาก server เพื่อเข้าถึง internal resource ที่เข้าจากข้างนอกไม่ได้ เช่น metadata service, internal API เป็นต้น
- **Root fix:** ควบคุมว่า server สามารถส่ง request ไปยังปลายทางใดได้บ้าง (allowlist + block internal IP ranges)
- **Ref:** CVE — Capital One breach (2019) — SSRF เข้า AWS metadata endpoint ได้ credentials ดึงข้อมูลลูกค้า 100 ล้านคน

**จุดที่มักเกิด SSRF:**
- Feature "Import from URL" / "Fetch image from URL"
- PDF generator ที่รับ URL
- Webhook ที่ user ตั้ง URL เอง

**จำแยก SSRF vs CSRF:**
- SSRF = หลอก **Server** ส่ง request (Server-Side)
- CSRF = หลอก **User** ส่ง request (Client-Side / Cross-Site)

---

## 2. XXE (XML External Entity Injection)

XXE คือการโจมตีผ่าน XML ที่ทำหน้าที่การเก็บข้อมูล ใช้ในการรับ-ส่งข้อมูล

- **Severity:** High
- **Root cause:** XML parser มีการเปิดการใช้งาน feature external entity
- **Impact:** attacker สามารถใช้งานช่องโหว่ดังกล่าวในการเข้าถึง internal resource ได้ผ่าน external entity
  1. อ่านไฟล์ใน server ผ่าน `file://` เช่น `/etc/passwd` เป็นต้น
  2. ยกระดับเป็น SSRF ผ่าน `http://` เข้า internal service
- **Root fix:** ปิดการใช้งาน feature external entity บน XML parser
- **Ref:** CVE-2014-3529 (Apache POI XXE) — library ที่ parse ไฟล์ Excel/Word (ซึ่งข้างในเป็น XML) ไม่ได้ปิด external entity ทำให้ attacker ส่งไฟล์ .xlsx ที่ฝัง XXE payload เข้าไปอ่านไฟล์ใน server ได้

**สิ่งที่ต้องจำ:**
- .xlsx, .docx ข้างในคือ ZIP → XML files → XML parser → XXE possible
- XXE เดี่ยวๆ ทำ RCE โดยตรงไม่ได้ แต่เป็นจุดเริ่มต้นของ attack chain (เช่น อ่าน config → ได้ credentials → เข้า database)

---

## 3. Security Misconfiguration

Security Misconfiguration คือการตั้งค่าระบบที่ไม่เหมาะสม ส่งผลให้เกิดช่องโหว่ เช่น default error page, default credentials, enable service/port unnecessary เป็นต้น

- **Severity:** High
- **Root cause:** การตั้งค่าระบบที่ไม่เหมาะสม ส่งผลให้เกิดช่องโหว่
- **Impact:** attacker สามารถนำข้อมูลดังกล่าวไปใช้งานต่อเพื่อสร้างช่องโหว่ที่ต้องการ
- **Root fix:** ปิด service/port ที่ไม่จำเป็น, เปลี่ยน default credentials, ปิด debug mode บน production, ลบ default page, ตั้ง permission ให้เหมาะสม, แก้ไข default error page
- **Ref:** CVE-2017-9805 (Apache Struts 2) — default config เปิดให้ deserialize XML จาก user input โดยไม่จำกัด attacker ส่ง payload เข้ามาทำ Remote Code Execution ได้

**ตัวอย่างที่พบบ่อย:**
- Default credentials (admin/admin)
- `info.php` (phpinfo) หลุดบน production — เปิดเผย PHP version, modules, server path
- Directory listing เปิดอยู่
- Permission 777 (จากประสบการณ์จริง)
- Apache/Nginx default welcome page บอก version

---

## 4. Using Components with Known Vulnerabilities

Using Components with Known Vulnerabilities คือการใช้งานส่วนประกอบในการพัฒนาที่ได้รับการแจ้งแล้วว่ามีช่องโหว่

- **Severity:** Critical
- **Root cause:** ระบบใช้ library/framework/component ที่มี CVE ประกาศแล้วแต่ยังไม่ได้ update หรือ patch
- **Impact:** attacker อาศัยส่วนประกอบนั้น เช่น library ที่เก่า, package ที่มีช่องโหว่ ในการโจมตีระบบ
- **Root fix:** dependency scanning อัตโนมัติ, Software Bill of Materials (SBOM) เพื่อให้รู้ว่าระบบใช้งานส่วนประกอบอะไรอยู่, มี policy กำหนด SLA ว่า Critical CVE ต้อง patch ภายในกี่วัน
- **Ref:** CVE-2021-44228 (Log4Shell) — Apache Log4j library ที่ Java application แทบทุกตัวใช้ มีช่องโหว่ที่ attacker สามารถทำ Remote Code Execution ได้แค่ส่ง string เข้ามาใน log กระทบทั่วโลกเพราะ Log4j ฝังอยู่ใน dependency chain

**ทำไม developer ไม่ patch:**
- กลัว patch แล้วระบบพัง (dependency chain)
- ไม่รู้ว่าใช้ library อะไรอยู่ (ไม่มี dependency tracking)
- Legacy system ที่ vendor เลิก support

---

## 5. Insufficient Logging & Monitoring

Insufficient Logging & Monitoring คือการบันทึกกิจกรรมที่เกิดขึ้นในระบบและแจ้งเตือนไม่เพียงพอ

- **Severity:** Critical
- **Root cause:** ระบบไม่มีการบันทึกกิจกรรมที่เกิดขึ้นในระบบและแจ้งเตือนที่เพียงพอ
- **Impact:** attacker สามารถโจมตีระบบได้ โดยที่ไม่มีการตรวจสอบข้อมูลอย่างทันท่วงที ส่งผลให้เกิดความเสียหาย
- **Root fix:** เก็บ log ให้ครอบคลุม + ตั้ง monitoring และ alerting เมื่อเจอ pattern ผิดปกติ
- **Ref:** Equifax breach (2017) — attacker อยู่ในระบบ 76 วันก่อนถูกตรวจพบ เพราะ monitoring ไม่เพียงพอ ข้อมูลลูกค้า 147 ล้านคนรั่ว ถ้าระบบตรวจจับได้เร็วกว่านี้ความเสียหายอาจจะจำกัดมากกว่านี้

**Log ที่ดีควรเก็บ:**
- **Who** — ใครทำ (user, IP, session)
- **What** — ทำอะไร (login, access file, change setting)
- **When** — เมื่อไหร่ (timestamp)
- **Where** — ที่ไหน (endpoint, resource)
- **Result** — สำเร็จหรือไม่ (success/fail)

**จำแยก Log vs Monitoring:**
- Log = บันทึกกิจกรรม เหตุการณ์ ช่วงเวลา
- Monitoring = นำ log มาวิเคราะห์ เฝ้าระวัง ประเมินผล และแจ้งเตือน
- มี log แต่ไม่มี monitoring = กล้องวงจรปิดที่ไม่มีคนดูจอ

---

*Day 12 Complete — ต่อไป: Day 13 Self-test OWASP 10 ข้อ*
