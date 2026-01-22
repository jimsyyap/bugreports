# Bug Bounty Engagement - Redacted Report

**Date:** January 22, 2026
**Target Ecosystem:** [Major Marketing SaaS Platform]
**Status:** Concluded (No critical vulnerabilities found)

---

## 1. Executive Summary

We conducted a deep-dive security assessment of the target ecosystem, focusing on three major asset groups (Core Platform + 2 Acquisitions).

**Key Conclusion:** The target assets demonstrate a hardened security posture. While we successfully identified "Soft Spots" (WAF-free endpoints) and mapped internal API structures, the application logic correctly handles authorization and input validation in all tested scenarios.

---

## 2. Methodology & Findings

### Campaign 1: [Acquisition A - PR Software]

**Goal:** Bypass the rigorous WAF/ACLs protecting tenant data.

* **Action:** Developed Custom Tooling for TLS Spoofing and Origin Shuffling.
* **Finding 1 (WAF Bypass):** Identified that specific tenants (`[REDACTED_TENANT].target-a.com`) were not protected by the primary WAF.
* **Finding 2 (Tenant Isolation):** Attempted Cross-Tenant IDOR (using Tenant A's origin to access Tenant B's data). Result: **Blocked (404)**.
* **Finding 3 (Local IDOR):** Mined sequential IDs using S3 Asset scrape. Fuzzed neighbors. Result: **Clean**.
* **Verdict:** **Secure**. Strong Logical Isolation compensates for inconsistent WAF coverage.

### Campaign 2: [Acquisition B - Competitive Intelligence]

**Goal:** Exploit the Single Page Application (SPA) architecture.

* **Action:** Executed Angular Bundle Analysis on `[REDACTED_PORTAL].target-b.com`.
* **Finding 1 (Tech Stack):** Angular Frontend hosted on Google Cloud. No WAF detected.
* **Finding 2 (Secrets):** Extracted Google API Key (`AIza[REDACTED]`) and Stripe Live Keys.
  * **Audit:** Key is restricted to **Maps/Places API**. Low Risk.
* **Finding 3 (API Access):** Fuzzed critical endpoints (`/action/health/createAccount`).
  * **Result:** **403 Forbidden**. Auth is correctly enforced.
* **Verdict:** **Secure**. Hardened Application Layer.

### Campaign 3: [Core Platform - Identity]

**Goal:** Compromise the Central Authentication Hub.

* **Action:** Fingerprinted `oauth.[TARGET].com` and deployed custom OAuth fuzzer.
* **Finding 1 (Attack Surface):** The OAuth portal is a React App on GCP with **No WAF**. XSS payloads returned 200 OK.
* **Finding 2 (Exploitation):** Stateless fuzzing of `redirect_uri`, `next`, `return_to`.
  * **Result:** **Clean**. The application validates redirect destinations against an allowlist and escapes Reflected parameters.
* **Verdict:** **Secure**.

---

## 3. Tooling Developed

The following custom scripts were created for this engagement:

| Script | Purpose |
| :--- | :--- |
| `soft_raider.py` | Exploits Soft Spots (WAF-free domains) to test tenant isolation. |
| `local_raider.py` | Regex-scrapes asset paths to find sequential IDs. |
| `ct_miner.py` | Certificate Transparency log miner for asset discovery. |
| `soft_param_miner.py` | Functionality miner (XSS/Configs) for soft targets. |
| `kdl_prober.py` | Fuzzer for internal APIs. |
| `oauth_miner.py` | Specialized fuzzer for OAuth Open Redirects and XSS. |
| `key_audit.py` | Audits API Keys against excessive permissions. |

---

## 4. Visual Proofs

[REDACTED: Visual Proofs removed for public disclosure compliance]
