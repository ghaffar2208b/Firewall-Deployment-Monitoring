# Deployment Report — SafeLine WAF

**Host:** kali (Kali GNU/Linux, VirtualBox VM, 2 CPU / 4GB RAM)
**Analyst:** Abdul Ghaffar
**Date:** 2026-07-22

---

## 1. Executive Summary

SafeLine WAF was successfully deployed on a resource-constrained single VM, configured as a reverse proxy in front of an intentionally vulnerable target application (DVWA). Deployment required resolving three distinct issues: a Docker repository incompatibility with Kali's rolling-release identifier, a container-networking misconfiguration causing 502 errors, and a resource-exhaustion crash of the target application. All issues were diagnosed and resolved; the full traffic chain (browser → WAF → application) was verified working end-to-end.

## 2. Scope

- Host: single Kali Linux VM
- Components deployed: Docker Engine, DVWA (target), SafeLine WAF (protection)
- Goal: verify WAF can intercept and inspect traffic to a real (if intentionally vulnerable) backend application

## 3. Deployment Timeline

| Step | Action | Outcome |
|---|---|---|
| 1 | Checked VM resources (1 CPU/1.9GB initially) | Insufficient for SafeLine's stated minimum (2 CPU/4GB); VM resources increased |
| 2 | Installed Docker via official install script | Failed — script assumed `kali-rolling` as a valid Docker repo codename |
| 3 | Manually configured Docker's Debian repo using `bookworm` codename | Resolved; Docker Engine 29.6.2 installed successfully |
| 4 | Deployed DVWA container (`vulnerables/web-dvwa`), port 8081 | Successful, confirmed via `curl` |
| 5 | Installed SafeLine WAF via official installer script | Successful after re-running with `sudo` (initial run failed on privilege check) |
| 6 | Retrieved admin credentials via `docker exec safeline-mgt resetadmin` | Successful — install summary was lost to a VM display freeze mid-install |
| 7 | Registered DVWA as a SafeLine "Application" (domain: localhost, port 80, upstream: 127.0.0.1:8081) | Resulted in 502 Bad Gateway |
| 8 | Diagnosed upstream networking issue | Root cause: 127.0.0.1 inside a container refers to itself, not the host |
| 9 | Corrected upstream to host's real IP (10.0.2.15:8081) | Full chain confirmed working |
| 10 | DVWA container found crashed (exit code 137) mid-testing | Root cause: OOM kill under combined resource load |
| 11 | Recreated DVWA with explicit resource limits (`--cpus=0.5 --memory=512m`) | Stabilized; no further crashes during remaining testing |

## 4. Evidence

| Evidence | Location |
|---|---|
| VM resource confirmation | screenshots/installation/ |
| Docker install/version confirmation | (terminal output, this report) |
| DVWA deployed | screenshots/installation/dvwa_deployed.png |
| SafeLine containers running | screenshots/installation/safeline_containers_running.png |
| SafeLine dashboard login | screenshots/installation/safeline_dashboard_login.png |
| Application creation form | screenshots/rules/safeline_add_application_form.png |
| Application created (Defense status) | screenshots/rules/safeline_application_created.png |
| 502 error (pre-fix) | (documented in this report and nginx error log excerpt below) |
| DVWA reachable via SafeLine (post-fix) | screenshots/rules/dvwa_via_safeline_success.png |
| DVWA resource-capped after crash | screenshots/installation/dvwa_resource_capped.png |

## 5. Commands Used

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh          # failed — kali-rolling repo not recognized
sudo rm /etc/apt/sources.list.d/docker.list
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian bookworm stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

docker run -d --name dvwa -p 8081:80 vulnerables/web-dvwa

sudo bash -c "$(curl -fsSLk https://waf.chaitin.com/release/latest/setup.sh)"
sudo docker exec safeline-mgt resetadmin

# Upstream fix
# SafeLine UI: Applications > DVWA-Test > Upstream changed from
# http://127.0.0.1:8081 to http://10.0.2.15:8081

# DVWA crash recovery
docker stop dvwa
docker rm dvwa
docker run -d --name dvwa -p 8081:80 --cpus="0.5" --memory="512m" vulnerables/web-dvwa
```

## 6. Findings

**Docker/Kali compatibility:** Docker's official install script does not recognize `kali-rolling` as a valid Debian codename for its repository, since Docker's repo is built against stable Debian release names. Kali being Debian-based but not a recognized Debian release causes the automated installer to fail at the repository-configuration step. This is resolved by manually pointing the Docker apt source at a real Debian codename (`bookworm`), which works because Kali maintains binary compatibility with Debian at the package level.

**Container networking (127.0.0.1 pitfall):** `127.0.0.1`/`localhost` inside a Docker container always refers to that container's own network namespace, never the host or other containers, unless containers share a network namespace explicitly. SafeLine's proxy container could not reach DVWA at `127.0.0.1:8081` because nothing was listening on that port *inside SafeLine's own container*. The fix was to use the VM's actual host interface IP address, which both containers can reach externally.

**Resource exhaustion:** Running SafeLine's 7 containers, DVWA (including its internal MySQL instance), a full XFCE desktop, and Firefox simultaneously on 2 CPU cores and ~4GB RAM exceeded available memory during testing, resulting in the Linux kernel's OOM killer terminating the DVWA container (exit code 137, SIGKILL). This also correlated with repeated VM input/display freezes throughout testing. Applying explicit container resource limits (`--cpus`, `--memory`) prevented DVWA specifically from contributing further to this problem.

## 7. Risk Assessment

| Finding | Severity | Notes |
|---|---|---|
| Docker repo incompatibility | Low | One-time setup friction, not a security risk |
| Upstream misconfiguration (502) | Low | Self-correcting via proper configuration; would prevent WAF from functioning if uncorrected in production |
| Resource exhaustion / OOM kills | Medium | In production, an under-provisioned WAF host could itself become an availability risk — the WAF or its protected app could crash under real traffic load exactly as DVWA did here under test load |

## 8. Recommendations

- Always verify Docker's official install script compatibility with non-standard/rolling Linux distributions before relying on it; be prepared to manually configure repositories against a compatible base distribution
- When deploying any reverse-proxy WAF, always use the host's actual routable IP for backend/upstream addresses — never assume `127.0.0.1`/`localhost` works across separate containers
- Size WAF deployment hosts with real headroom above the vendor's stated minimum, especially when the same host is also running the protected application and any desktop/testing tools simultaneously
- Apply explicit container resource limits to all workloads on constrained hosts to prevent one runaway container from destabilizing the entire system

## 9. Lessons Learned

Two of the three deployment issues (Docker/Kali repo mismatch, container networking) were pure environment/configuration problems solvable through standard Linux/Docker troubleshooting methodology: read the actual error, form a hypothesis, test it directly (e.g., `curl` from inside the container to isolate exactly where connectivity broke), then fix the root cause rather than a workaround. The third issue (resource exhaustion) reinforced that security tooling has real infrastructure requirements — a WAF that crashes under load provides zero protection, so capacity planning is itself a part of a secure deployment, not a separate concern.
