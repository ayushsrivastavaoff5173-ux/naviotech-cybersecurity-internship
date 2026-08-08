# Zero Trust Simulation — Setup & Testing Guide

This folder contains a ready-to-run simulation of the Zero Trust framework described in the
project report. It uses Docker to represent three isolated enterprise "segments" (Finance, HR,
Engineering) behind a policy-enforcement gateway, plus Keycloak as the enterprise IAM/MFA
provider.

**Important:** This must be run on your own laptop/PC (not in this chat) because it needs Docker
Desktop and internet access to pull container images. Follow the steps below, then take the
screenshots listed at the end — those screenshots go into your report and GitHub repo.

---

## 1. Prerequisites

- Install **Docker Desktop** (Windows/Mac) or **Docker Engine + Docker Compose** (Linux):
  https://www.docker.com/products/docker-desktop/
- Confirm it works:
  ```bash
  docker --version
  docker compose version
  ```

## 2. Start the simulation

From inside this `simulation/` folder:

```bash
docker compose up -d
```

Wait ~30-60 seconds for Keycloak to fully start. Check everything is running:

```bash
docker compose ps
```

You should see 5 containers: `zt-keycloak`, `zt-finance-app`, `zt-hr-app`,
`zt-engineering-app`, `zt-gateway`.

**📸 Screenshot 1:** Terminal output of `docker compose ps` showing all 5 containers running.

## 3. Test the Zero Trust Gateway (Policy Enforcement Point)

The gateway is available at `http://localhost:8081`. It only allows a request through to a
segment if the correct `X-Role` header is present — simulating the role claim that would
normally come from a verified Keycloak JWT.

**Test A — No role header (should be BLOCKED):**
```bash
curl -i http://localhost:8081/finance/
```
Expected: `403 Access Denied`

**📸 Screenshot 2:** This blocked response.

**Test B — Correct role (should be ALLOWED):**
```bash
curl -i -H "X-Role: finance" http://localhost:8081/finance/
```
Expected: `200 OK` with the Finance Segment welcome page.

**📸 Screenshot 3:** This successful response (or open it in a browser using a header-editing
extension, and screenshot the rendered page).

**Test C — Lateral movement attempt (should be BLOCKED):**
```bash
curl -i -H "X-Role: finance" http://localhost:8081/hr/
```
A Finance-role caller trying to reach the HR segment must still be denied.

**📸 Screenshot 4:** This blocked response — this is your "insider threat" test proving
micro-segmentation works even with a valid, authenticated role.

**Test D — IT Admin (should be ALLOWED everywhere):**
```bash
curl -i -H "X-Role: it-admin" http://localhost:8081/finance/
curl -i -H "X-Role: it-admin" http://localhost:8081/hr/
curl -i -H "X-Role: it-admin" http://localhost:8081/engineering/
```

**📸 Screenshot 5:** All three succeeding for the admin role.

## 4. Configure Keycloak (Identity Verification + MFA)

1. Open http://localhost:8080 in your browser.
2. Log in with `admin` / `admin123`.
3. Create a new Realm, e.g. `naviotech-zta`.
4. Under **Realm roles**, create: `employee`, `finance`, `hr`, `engineer`, `it-admin`
   (matching the RBAC matrix in the report).
5. Under **Users**, create a test user (e.g. `test.finance`), assign the `finance` role.
6. Log in as that user at the realm's account console
   (`http://localhost:8080/realms/naviotech-zta/account`) and set up **TOTP-based MFA**
   using an authenticator app (Google Authenticator / Authy).

**📸 Screenshot 6:** Keycloak realm roles list.
**📸 Screenshot 7:** MFA/TOTP setup screen (QR code) during first login.
**📸 Screenshot 8:** Successful login after entering the MFA code.

## 5. Verify network-level isolation (optional, for extra credit)

Show that segments truly sit on separate Docker networks and cannot reach each other directly:

```bash
docker network ls
docker network inspect simulation_seg_finance
```

Notice `zt-hr-app` and `zt-engineering-app` are **not** listed as members of the
`seg_finance` network — this is the container-level equivalent of micro-segmentation / VLANs
in a real enterprise network.

**📸 Screenshot 9:** `docker network inspect` output showing only `zt-finance-app` and
`zt-gateway` attached to the Finance network.

## 6. Stop the simulation

```bash
docker compose down
```

---

## Where these screenshots go

Save all screenshots into `simulation/screenshots/` in this same structure, then reference them
in **Section 8 (Security Testing & Results)** of `Minor_Project_2_Report.docx` and in your
GitHub repository's README, as required by the internship submission instructions.

## How this maps to the report

| Simulation element | Report section |
|---|---|
| `docker-compose.yml` network layout | Section 5 — Framework Design (micro-segmentation) |
| `nginx/nginx.conf` role checks | Section 5 — Policy Enforcement Point |
| Keycloak realm/roles/MFA | Section 7 — IAM Implementation |
| Screenshots 2-5 | Section 8 — Security Testing & Results (test table) |
| Screenshots 6-9 | Section 8 — Security Testing & Results (evidence) |
