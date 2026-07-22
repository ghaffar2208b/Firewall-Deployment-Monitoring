# Firewall Deployment & Monitoring — SafeLine WAF

A hands-on Web Application Firewall (WAF) deployment lab, built entirely on a single Kali Linux VM. This project deploys SafeLine WAF in front of an intentionally vulnerable web application (DVWA), then launches real attacks from the same machine to test, monitor, and document what the firewall actually catches.

Everything in this repository is real: real containers, real attacks, real blocks, and real troubleshooting — including genuine environment failures encountered and resolved along the way.

---

## 1. Project Overview

A Web Application Firewall sits between users and a web app, inspecting every request before it reaches the backend. This lab builds that setup end-to-end on constrained hardware (a single 2-core/4GB VM) to demonstrate:

- Deploying a reverse-proxy WAF (SafeLine) in front of a vulnerable target (DVWA)
- Configuring protection rules: signature-based attack detection, rate limiting
- Launching real attack simulations: SQL injection, XSS, HTTP flood, brute-force login
- Monitoring and interpreting blocked-connection evidence from the WAF's own logs
- Documenting findings — including a genuine limitation discovered in the product's default configuration

## 2. Objectives

1. Deploy Docker, a vulnerable target application, and a WAF entirely on one VM
2. Correctly configure a reverse-proxy WAF application (domain, port, upstream)
3. Confirm and read WAF protection module configuration (Semantic Analysis, HTTP Flood, Auth)
4. Launch real attacks and observe/interpret the WAF's detection and blocking behavior
5. Diagnose and resolve real environment failures (networking misconfiguration, resource exhaustion, container crashes)
6. Document findings as a professional deployment + incident-style report set

## 3. Environment & Tools

| Component | Detail |
|---|---|
| Host | Kali Linux VM (VirtualBox), 2 CPU / 4GB RAM |
| Containerization | Docker Engine 29.6.2, Docker Compose v5.3.1 |
| Target application | DVWA (Damn Vulnerable Web Application), `vulnerables/web-dvwa`, port 8081 |
| WAF | SafeLine WAF (Chaitin Tech), version 9.3.10, 7-day trial |
| WAF management | `https://localhost:9443` |
| Protected endpoint | `http://localhost` (port 80, proxied to DVWA) |

## 4. Architecture

Attacker/Browser (same VM)
│
▼
http://localhost:80
│
┌────▼─────────────┐
│ SafeLine WAF │ (reverse proxy + detection engine)
│ safeline-tengine │
└────┬─────────────┘
│ upstream: http://10.0.2.15:8081
▼
┌───────────────┐
│ DVWA (Docker) │
│ port 8081 │
└───────────────┘

Both SafeLine and DVWA run as separate Docker containers on the same host. SafeLine's proxy container uses Docker's `host` network mode, so it reaches DVWA via the VM's actual network interface IP (`10.0.2.15`) rather than `127.0.0.1`, which — inside a container — refers to the container itself, not the host.

## 5. Folder Structure

03-Firewall-Deployment-Monitoring/
├── README.md
├── notes/
│ ├── waf_fundamentals.md
│ └── attack_simulation_log.md
├── reports/
│ ├── deployment_report.md
│ ├── attack_simulation_report.md
│ └── blocked_connections_report.md
├── screenshots/
│ ├── installation/ dashboard/ rules/ attacks/ logs/
└── exports/


## 6. Deployment Summary

- Docker Engine installed manually via Docker's official Debian repo (Kali's `docker-cli`/`podman-docker` packages and Docker's `kali-rolling` auto-detection were both non-viable; resolved by pointing at Debian `bookworm`'s repo instead)
- DVWA deployed via Docker, exposed on host port 8081
- SafeLine WAF deployed via its official installer script, exposing the management console on port 9443
- DVWA registered as a SafeLine "Application": domain `localhost`, port 80, reverse proxy, upstream `http://10.0.2.15:8081`
- Full chain verified: browser → SafeLine (port 80) → DVWA (port 8081)

Full troubleshooting narrative (including a `127.0.0.1` vs. host-IP networking issue and a DVWA container crash from resource exhaustion) is documented in `reports/deployment_report.md`.

## 7. Attack Simulations & Results

| # | Attack Type | Method | Result |
|---|---|---|---|
| 1 | SQL Injection | `' OR '1'='1` in DVWA's SQLi page | **Blocked** — SafeLine Semantic Analysis, module: SQL Inj |
| 2 | Cross-Site Scripting (XSS) | `<script>alert('XSS')</script>` in DVWA's reflected XSS page | **Blocked** — SafeLine Semantic Analysis, module: XSS |
| 3 | HTTP Flood | 150 concurrent requests to `/` | **Blocked** — Basic Access Limit (100 reqs/10s), Anti-Bot Challenge issued |
| 4 | Brute-force login | 10 rapid failed POSTs to `/login.php` | **Not blocked** — finding documented below |

Full evidence, payloads, and forensic detail for each are in `reports/attack_simulation_report.md` and `reports/blocked_connections_report.md`.

## 8. Key Finding: Brute-Force / Login Protection Gap

SafeLine's **Auth** module is a Single Sign-On (SSO) tracking feature, not an automatic brute-force detector for arbitrary backend login forms. Since DVWA's login page isn't integrated with SafeLine's SSO layer, 10 consecutive failed login attempts passed through untouched. Real brute-force protection for a custom login endpoint like DVWA's would require either:
- A custom rate-limit rule scoped specifically to the `/login.php` path, or
- SafeLine's "Basic Attack Limit" rule (blocks after repeated *detected attacks*, not failed logins specifically)

This is documented as a genuine configuration gap discovered during testing, not a product failure — and reflects a real lesson: **default WAF rules protect against generic attack patterns; protecting a specific business-logic endpoint (like a login form) usually requires an explicit custom rule.**

## 9. Skills Learned

- Deploying and networking a multi-container Docker security stack on a single host
- Configuring a reverse-proxy WAF (domain, port, upstream) and diagnosing container-to-container networking issues
- Reading and interpreting WAF signature-based detection modules (SQLi, XSS) and rate-limiting modules
- Running real attack simulations and validating firewall behavior from evidence, not assumption
- Diagnosing resource-exhaustion failures (OOM-killed containers) and applying container resource limits as remediation
- Recognizing the difference between a product default and a genuine security gap requiring custom configuration

## 10. Future Improvements

- Configure a custom rate-limit rule scoped to `/login.php` and re-test the brute-force scenario to close the identified gap
- Add a second attack source (a different container or, resources permitting, a second VM) to simulate a non-localhost attacker
- Enable SafeLine's Bot Protection and Anti-Bot modules and test against automated scanning tools (e.g., `nikto`)
- Upgrade past the 7-day trial or explore SafeLine's open-source tier for persistent log export

## 11. License

This project is released under the MIT License.

---

*All findings in this repository were generated through real attack simulations against a self-hosted, intentionally vulnerable application on the author's own VM. No fabricated logs or results are included.*
