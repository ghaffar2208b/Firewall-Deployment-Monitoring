# Attack Simulation Report

**Host:** kali (Kali GNU/Linux VM)
**Target:** DVWA behind SafeLine WAF (`http://localhost`)
**Analyst:** Abdul Ghaffar
**Date:** 2026-07-22

---

## 1. Executive Summary

Four attack categories were simulated against a SafeLine-protected DVWA instance: SQL injection, reflected XSS, HTTP flood, and brute-force login. Three of the four were successfully detected and blocked by SafeLine's default configuration (SQLi, XSS, HTTP Flood). The fourth (brute-force login) was not blocked, and investigation determined this was due to a configuration gap rather than a WAF failure — SafeLine's built-in Auth module is an SSO-tracking feature, not a generic login-brute-force detector, and no custom rate-limit rule had been configured for the specific login endpoint.

## 2. Scope

- Target application: DVWA, security level "Low"
- Protected endpoint: `http://localhost` (SafeLine reverse proxy)
- Attack source: same VM (127.0.0.1 / LAN IP), via browser and terminal (curl)

## 3. Timeline

| Time | Attack | Result |
|---|---|---|
| 10:25:41 | SQL Injection (`' OR '1'='1`) | Blocked |
| 10:40:12 | XSS (`<script>alert('XSS')</script>`) | Blocked |
| 16:04:15 | HTTP Flood (150 concurrent requests) | Blocked (Anti-Bot Challenge, 60 min) |
| (following) | Brute-force login (10x failed POST to /login.php) | Not blocked |

## 4. Evidence

| Evidence | Location |
|---|---|
| SQLi block page | screenshots/attacks/sqli_blocked_by_safeline.png |
| SQLi forensic detail | screenshots/logs/sqli_full_forensic_detail.png |
| XSS block page | screenshots/attacks/xss_blocked_by_safeline.png |
| XSS forensic detail | screenshots/logs/xss_full_forensic_detail.png |
| HTTP Flood rate-limit trigger | screenshots/attacks/http_flood_rate_limit_triggered.png |
| Auth module "NoData" (brute-force gap) | screenshots/rules/safeline_auth_module_nodata.png |
| HTTP Flood settings (thresholds) | screenshots/rules/safeline_http_flood_settings.png |

## 5. Commands / Payloads Used

**SQL Injection:** `' OR '1'='1` (submitted via DVWA SQLi form)

**XSS:** `<script>alert('XSS')</script>` (submitted via DVWA reflected XSS form)

**HTTP Flood:**
```bash
for i in {1..150}; do curl -s -o /dev/null -w "%{http_code}\n" http://localhost/ & done; wait
```

**Brute-force login:**
```bash
for i in {1..10}; do curl -s -o /dev/null -w "%{http_code}\n" -X POST "http://localhost/login.php" -d "username=admin&password=wrongpass$i&Login=Login" -c /tmp/cookies.txt -b /tmp/cookies.txt; done
```

## 6. Findings

**SQL Injection:** SafeLine's Semantic Analysis engine (SQL Inj module, Balance Mode) correctly identified and blocked the classic tautology-based injection payload on first attempt, returning an "Access Forbidden" page with a unique traceable event ID (d411383f5da64e13b2e7b35267763edd) instead of forwarding the request to DVWA.

**XSS:** The same Semantic Analysis engine (XSS module, Balance Mode) correctly identified and blocked a script-tag-based reflected XSS payload on first attempt (event ID 5c9c3a1e974d4e96958d84750157b9e2), again returning "Access Forbidden" rather than reflecting the payload back through DVWA.

**HTTP Flood:** SafeLine's "Basic Access Limit" rule (100 requests within 10 seconds → Anti-Bot Challenge for 60 minutes) was NOT enabled by default; the rule existed but was toggled off. A sequential 50-request test also failed to trigger it purely due to request-rate limitations of sequential curl calls (too slow to cross the 10-second window threshold). Once the rule was manually enabled and a concurrent 150-request burst was sent, the threshold was crossed and the source IP was correctly challenged.

**Brute-force login:** 10 rapid, distinct failed-password POST requests against DVWA's actual login endpoint were not detected or blocked by any SafeLine module. Investigation of the "Auth" module confirmed it tracks SSO-integrated logins only, and DVWA's login form is not SSO-integrated, so this traffic pattern was invisible to that module. Neither of SafeLine's other rate-limiting rules (Basic Access Limit, Basic Attack Limit) are scoped to match this specific pattern (10 requests is under the Access Limit's 100-request threshold, and no "attack" was detected to trigger the Attack Limit rule).

## 7. Risk Assessment

| Finding | Severity | Likelihood | Risk |
|---|---|---|---|
| SQLi/XSS successfully blocked | N/A (mitigation confirmed) | N/A | Low — core protections functioning as intended |
| HTTP Flood rule disabled by default | Medium | High (default state) | Medium — a real deployment left at default settings would be vulnerable to flood/DoS traffic until manually enabled |
| No brute-force protection for custom login endpoint | High | High (default state, common real-world pattern) | High — a real login form behind this WAF configuration would remain vulnerable to credential-stuffing/brute-force attacks without additional custom rule configuration |

## 8. Recommendations

- Explicitly enable and review all HTTP Flood rules after any fresh SafeLine deployment — do not assume rate-limiting is active by default
- Configure a custom rate-limit rule scoped specifically to authentication endpoints (e.g., `/login.php`) with a low threshold (e.g., 5 failed attempts per minute per IP), since generic rules will not catch this pattern
- Treat SafeLine's "Auth" module as an SSO feature, not a substitute for endpoint-specific brute-force protection, when auditing a deployment's actual security posture
- Periodically re-run these same attack simulations after any configuration change, to regression-test that protections remain effective

## 9. Lessons Learned

Not every protection in a WAF is active out of the box, and a feature's *name* does not always match its actual function — "Auth" sounded like it should catch failed logins, but its real purpose (SSO tracking) only became clear by checking its actual data output and reasoning through the empty result rather than assuming the module was simply broken. Confirming a security control's real behavior through direct testing, rather than trusting a dashboard label, is a core habit this exercise reinforced.
