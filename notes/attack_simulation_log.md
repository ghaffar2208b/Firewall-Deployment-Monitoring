# Attack Simulation Log

Chronological log of every attack simulation run in this lab, with raw commands and outcomes. See `reports/attack_simulation_report.md` for the full analytical writeup.

---

## 1. SQL Injection

**Time:** 2026-07-22 10:25:41 EDT (approx, per SafeLine log)
**Target:** `http://localhost/vulnerabilities/sqli/`
**Method:** Manual form submission via browser
**Payload:** `' OR '1'='1`
**Full triggered URL:** `http://localhost/vulnerabilities/sqli/?id='+OR+'1'='1&Submit=Submit`

**Result:** Blocked. SafeLine returned "Access Forbidden" page.
**Event ID:** d411383f5da64e13b2e7b35267763edd
**Detected module:** SQL Inj
**Detected payload location:** URLPATH
**Detected payload:** `\' OR \'1\'=\'1`

---

## 2. Cross-Site Scripting (XSS)

**Time:** 2026-07-22 10:40:12 EDT
**Target:** `http://localhost/vulnerabilities/xss_r/`
**Method:** Manual form submission via browser
**Payload:** `<script>alert('XSS')</script>`
**Full triggered URL:** `http://localhost/vulnerabilities/xss_r/?name=<script>alert('XSS')</script>`

**Result:** Blocked. SafeLine returned "Access Forbidden" page.
**Event ID:** 5c9c3a1e974d4e96958d84750157b9e2
**Detected module:** XSS
**Detected payload:** `<script>alert(\'XSS\')</script>`

---

## 3. HTTP Flood

**Time:** 2026-07-22 16:04:15 (per SafeLine Rate Limiting log)
**Target:** `http://localhost/`
**Method:** Terminal — concurrent curl loop

Command used:
```bash
for i in {1..150}; do curl -s -o /dev/null -w "%{http_code}\n" http://localhost/ & done; wait
```

**Result:** Basic Access Limit triggered — "100 Reqs within 10 seconds" threshold crossed.
**Action taken by SafeLine:** Anti-Bot Challenge issued to source IP for 60 minutes.
**Source IP:** 127.0.0.1 (LAN IP)

Note: An initial sequential (non-concurrent) 50-request test using a plain for-loop did NOT trigger this rule, because sequential curl calls are too slow (process-spawn overhead) to exceed the 10-second window threshold. Switching to concurrent (backgrounded `&`) requests was necessary to generate genuine burst volume.

---

## 4. Brute-Force Login Attempt

**Time:** 2026-07-22 (approx, immediately following flood test)
**Target:** `http://localhost/login.php`
**Method:** Terminal — sequential curl POST loop

Command used:
```bash
for i in {1..10}; do curl -s -o /dev/null -w "%{http_code}\n" -X POST "http://localhost/login.php" -d "username=admin&password=wrongpass$i&Login=Login" -c /tmp/cookies.txt -b /tmp/cookies.txt; done
```

**Result:** All 10 requests returned HTTP 302 (DVWA's standard failed-login redirect). No SafeLine block triggered.

**Investigation:** Checked SafeLine's "Auth" module (`localhost:9443/auth`) — returned "NoData." Determined this module is an SSO-tracking feature, not a generic brute-force detector, and does not monitor arbitrary backend login forms like DVWA's unless explicitly integrated.

**Conclusion:** This is a genuine configuration gap for this specific use case, not a WAF malfunction. Documented as a finding and future improvement (custom rate-limit rule needed for `/login.php`).

---

## Environment Issues Encountered During Testing

For completeness — these were real operational issues resolved during the lab, relevant to the deployment report:

1. **Upstream misconfiguration:** Initial SafeLine upstream was set to `http://127.0.0.1:8081`, causing 502 Bad Gateway errors. Root cause: `127.0.0.1` inside a container refers to the container itself, not the host. Fixed by using the VM's actual interface IP (`10.0.2.15`).
2. **DVWA container crash (exit code 137):** DVWA container was killed (SIGKILL), consistent with an OOM (out-of-memory) event under combined load from SafeLine's 7 containers + DVWA + desktop environment on a 2-core/~4GB VM. Resolved by recreating the container with explicit resource limits: `--cpus="0.5" --memory="512m"`.
3. **VM input/display freezes:** Multiple episodes of mouse/keyboard input becoming unresponsive, correlated with high combined CPU load from Docker + desktop + browser. Resolved via VM restarts; mitigated going forward by capping DVWA's resource usage.
