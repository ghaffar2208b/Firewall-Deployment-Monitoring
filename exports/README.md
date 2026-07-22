# Exports

- `safeline_nginx_error.log` — SafeLine's internal nginx error log, showing the
  upstream misconfiguration (127.0.0.1 vs. host IP) encountered and resolved
  during deployment. See reports/deployment_report.md, Section 6.

- Attack/traffic log export (SQLi, XSS, HTTP Flood events) was not available
  as a downloadable file: SafeLine's built-in log-export feature is a paid-tier
  capability, and this lab used the 7-day free trial. Full forensic detail for
  every attack (attack type, exact payload, timestamp, event ID, raw HTTP
  request) was instead captured directly from the dashboard's "Detail" view via
  screenshot, and is documented in:
    - reports/attack_simulation_report.md
    - reports/blocked_connections_report.md
    - screenshots/logs/
