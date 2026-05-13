# OWASP Top 10 Cheatsheet — Summary & Hands-on Labs

This self-study resource covers the OWASP Top 10 topics and includes practical exercises from TryHackMe. This document was created during a 30-day cybersecurity education program and serves as a review to reinforce understanding of previous material. Each topic includes: vulnerability descriptions, remediation guidance, misconceptions and correct answers, step-by-step challenge instructions, findings, and important lessons.

> **Note:** Study notes are written in Thai with English technical terms.

---

## OWASP Top 10 — Quick Summary

| # | Category | Description |
|---|----------|-------------|
| A01 | Broken Access Control | Access to restricted areas due to missing or broken authorization checks |
| A02 | Cryptographic Failures | Unprotected or inadequately protected sensitive data |
| A03 | Injection | Malicious commands injected into the target system via unsanitized input |
| A04 | Insecure Design | Poorly designed architecture resulting in fundamental security flaws |
| A05 | Security Misconfiguration | Poor configuration exposing the system to attacks |
| A06 | Vulnerable & Outdated Components | Using components with known vulnerabilities that remain unpatched |
| A07 | Identification & Authentication Failures | Broken authentication allowing unauthorized access |
| A08 | Software & Data Integrity Failures | Trusting unverified software updates or data without integrity checks |
| A09 | Security Logging & Monitoring Failures | Attacks go undetected due to insufficient logging and monitoring |
| A10 | SSRF | Server is tricked into making requests to internal resources on behalf of the attacker |

---

## Hands-on Labs

| Day | Topic | OWASP | TryHackMe Room |
|-----|-------|-------|----------------|
| Day 1-7 | SQL Injection (7 variants: First-Order, Second-Order, UNION-based, Boolean Blind, Chained Queries, Python Automation) | A03 | [SQLi Lab](https://tryhackme.com/room/sqlilab) |
| Day 8 | XSS (Reflected, Stored, DOM-based) | A03 | [XSS](https://tryhackme.com/room/axss) |
| Day 9 | CSRF (Hidden Link, Double Submit Cookie Bypass, SameSite Lax) | A01 | [CSRF](https://tryhackme.com/room/csrfV2) |
| Day 3, 10 | Broken Authentication (Session Fixation, Weak Lockout, MFA Bypass) | A07 | [Authentication Bypass](https://tryhackme.com/room/authenticationbypass), [MFA](https://tryhackme.com/room/multifactorauthentications) |
| Day 11 | IDOR (Parameter Tampering, Encoded/Hashed/Unpredictable IDs) | A01 | [IDOR](https://tryhackme.com/room/idor) |
| Day 12 | OWASP Quick Reference (SSRF, XXE, Misconfiguration, Vulnerable Components, Logging) | A03, A05, A06, A09, A10 | [SSRF](https://tryhackme.com/room/ssrfqi) |

---

## Tools Used

| Tool | Usage |
|------|-------|
| Burp Suite | Proxy, Repeater, Intruder — intercept and modify HTTP requests for testing |
| Nmap | Network scanning and service version detection |
| ffuf | Directory and parameter fuzzing |
| Python (requests) | Scripting automated SQL Injection payloads |

---

## Study Notes

- [Day 1 — SQL Injection Part 1](notes/Day_1_SQL_Injection_Part_1.md)
- [Day 2 — SQL Injection Part 2](notes/Day_2_SQL_Injection_Part_2.md)
- [Day 3 — Broken Authentication](notes/Day_3_Broken_Authentication.md)
- [Day 4 — Second-Order SQL Injection Part 1](notes/Day_4_Second_Order_SQL_Injection.md)
- [Day 5 — Second-Order SQL Injection Part 2](notes/Day_5_Second_Order_SQLi_Change_Password.md)
- [Day 6 — SQL Injection First-Order Part 1](notes/Day_6_Book_Title_SQLi.md)
- [Day 7 — SQL Injection First-Order Part 2](notes/Day_7_Book_Title_2_SQLi.md)
- [Day 8 — XSS (Cross-Site Scripting)](notes/Day_8_XSS.md)
- [Day 9 — CSRF (Cross-Site Request Forgery)](notes/Day_9_CSRF.md)
- [Day 10 — Broken Authentication Expanded](notes/Day_10_Broken_Auth.md)
- [Day 11 — IDOR (Insecure Direct Object Reference)](notes/Day_11_IDOR.md)
- [Day 12 — OWASP Quick Reference](notes/Day_12_OWASP_Quick_Reference.md)
