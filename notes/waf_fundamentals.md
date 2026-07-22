# WAF Fundamentals

## What is a WAF?

A Web Application Firewall (WAF) is a reverse proxy that sits between clients and a web application, inspecting HTTP/HTTPS traffic at the application layer (Layer 7) rather than just the network layer. Unlike a traditional network firewall — which filters by IP/port — a WAF understands HTTP requests well enough to detect malicious *content*: SQL injection strings, XSS payloads, abnormal request rates, malformed uploads, and more.

## Why put a WAF in front of an app instead of just patching the app?

In a real environment, patching every vulnerability in application code takes time, and some vulnerabilities are unknown until exploited (zero-days). A WAF provides a compensating control: it can block a known attack *pattern* even if the underlying application code is still vulnerable. This is exactly why DVWA (deliberately full of unpatched vulnerabilities) was still successfully protected against SQLi and XSS in this lab — the WAF caught the malicious pattern before it ever reached DVWA's vulnerable code.

## How SafeLine WAF is architected

SafeLine deploys as a set of Docker containers:

| Container | Role |
|---|---|
| `safeline-tengine` | The actual reverse-proxy/traffic engine (based on Tengine, an Nginx fork) |
| `safeline-detector` | Detection engine — semantic analysis for SQLi/XSS/etc. |
| `safeline-mgt` | Management web console (the dashboard UI) |
| `safeline-postgres` | Database backing the management console (config, logs metadata) |
| `safeline-chaos`, `safeline-fvm`, `safeline-luigi` | Supporting internal services |

Traffic flow: **Client → tengine (proxy) → detector (inspects) → decision (allow/block) → upstream backend (if allowed)**.

## Detection approach: Semantic Analysis

Rather than simple string/regex matching alone, SafeLine's "Semantic Analysis" engine parses request content to understand its structure (e.g., recognizing SQL syntax patterns, not just keyword matches), which reduces both false negatives (attacks that slip past naive filters) and false positives (legitimate content wrongly blocked). Each module (SQL Inj, XSS, File Uploading, File Including, CMD Inj, Java Code Inj) can be independently set to:

- **Disabled** — no inspection
- **Audit Mode** — logs matches but does not block
- **Balance Mode** — blocks confidently-malicious patterns (used in this lab)
- **Strict Mode** — most aggressive blocking, higher false-positive risk

## Rate limiting vs. content detection

Content-based detection (SQLi/XSS modules) and rate-based detection (HTTP Flood module) are separate protection layers solving different problems:
- **Content detection** catches a single malicious *request*, regardless of frequency.
- **Rate limiting** catches abnormal *volume* of otherwise-valid-looking requests — the pattern of a DoS attempt, credential-stuffing, or scraping, even if no individual request looks malicious on its own.

This distinction mattered directly in this lab: our SQLi/XSS payloads were caught by content detection instantly (one request each), while our flood test needed to cross a volume threshold (100 requests/10 seconds) before triggering a block — and our brute-force login test crossed neither threshold, because it was neither a recognized attack signature nor a high enough request volume in the tested window.

## Key lesson from this lab

A WAF's default configuration protects against *generic*, well-known attack classes out of the box (SQLi, XSS). It does **not** automatically understand your application's specific business logic (e.g., "this endpoint is a login form and should be rate-limited per-account") unless you explicitly configure a rule for it. This is why our brute-force test against DVWA's custom `/login.php` endpoint passed through untouched — a real deployment would need a custom rule targeting that specific path.
