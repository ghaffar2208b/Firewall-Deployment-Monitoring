# Blocked Connections Report

**Host:** kali (Kali GNU/Linux VM)
**WAF:** SafeLine WAF v9.3.10
**Analyst:** Abdul Ghaffar
**Date:** 2026-07-22

---

## 1. Executive Summary

This report catalogs every connection blocked by SafeLine WAF during attack simulation testing, with full forensic detail extracted directly from SafeLine's Attacks and HTTP Flood dashboards. Two distinct blocking mechanisms were observed: content-based detection (Semantic Analysis, blocking on payload signature) and volume-based detection (Rate Limiting, blocking on request frequency).

## 2. Blocked Connection #1 — SQL Injection

| Field | Value |
|---|---|
| Action | Blocked |
| Attack Type / Module | SQL Inj |
| Source IP | 127.0.0.1 (LAN IP) |
| Target URL | `http://localhost/vulnerabilities/sqli/?id=%27+OR+%271%27%3D%271&Submit=Submit` |
| Payload location | URLPATH |
| Detected payload | `\' OR \'1\'=\'1` |
| Timestamp | 2026-07-22 10:25:41 |
| Event ID | d411383f5da64e13b2e7b35267763edd |
| User-Agent | Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0 |

## 3. Blocked Connection #2 — Cross-Site Scripting (XSS)

| Field | Value |
|---|---|
| Action | Blocked |
| Attack Type / Module | XSS |
| Source IP | 127.0.0.1 (LAN IP) |
| Target URL | `http://localhost/vulnerabilities/xss_r/?name=%3Cscript%3Ealert%28%27XSS%27%29%3C%2Fscript%3E` |
| Payload location | URLPATH |
| Detected payload | `<script>alert(\'XSS\')</script>` |
| Timestamp | 2026-07-22 10:40:12 |
| Event ID | 5c9c3a1e974d4e96958d84750157b9e2 |

## 4. Blocked Connection #3 — HTTP Flood / Rate Limit

| Field | Value |
|---|---|
| Action | Blocked (Anti-Bot Challenge issued) |
| Rule | Basic Access Limit |
| Source IP | 127.0.0.1 (LAN IP) |
| Application | DVWA-Test (localhost) |
| Trigger condition | 100 requests within 10 seconds |
| Action duration | 60 minutes (Anti-Bot Challenge required for further access) |
| Timestamp | 2026-07-22 16:04:15 |

## 5. Non-Blocked Traffic — Brute-Force Login Attempt (documented for completeness)

| Field | Value |
|---|---|
| Action | Not blocked (all 10 requests returned HTTP 302) |
| Target | `http://localhost/login.php` |
| Source IP | 127.0.0.1 |
| Reason not blocked | No SafeLine module currently scoped to detect this pattern (see attack_simulation_report.md, Section 6) |

## 6. Summary Table

| # | Type | Detection Method | Action | Response Time to Block |
|---|---|---|---|---|
| 1 | SQL Injection | Content signature (Semantic Analysis) | Immediate block | 1st request |
| 2 | XSS | Content signature (Semantic Analysis) | Immediate block | 1st request |
| 3 | HTTP Flood | Volume threshold (Rate Limiting) | Anti-Bot Challenge | After ~100th request within 10s |
| 4 | Brute-force login | None configured | Not blocked | N/A |

## 7. Recommendations

- Content-signature-based rules (SQLi, XSS) demonstrated immediate, reliable blocking and required no additional configuration beyond the product default — no action needed here beyond periodic regression testing
- Volume-based rules require explicit enablement post-deployment; add this to any standard SafeLine deployment checklist
- Endpoint-specific abuse patterns (like login brute-forcing) require a dedicated custom rule; add creation of such a rule to deployment checklists for any application with an authentication endpoint

## 8. Lessons Learned

Structured log detail (attack type, exact payload, timestamp, and unique event ID) made it possible to build a precise, evidence-backed record of each block without needing the product's paid log-export feature — the in-dashboard "Detail" view provided everything needed for full forensic documentation via direct inspection and screenshot capture.
