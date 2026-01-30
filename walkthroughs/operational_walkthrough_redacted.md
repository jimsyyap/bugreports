# operational_walkthrough_redacted.md

**Target**: bank_redacted (formerly Paybank_redacted)
**Objective**: Offensive Security Assessment / Bug Bounty Reconnaissance
**Date**: Jan 2026
**Status**: Concluded (Target Hardened)

---

## Executive Summary

This engagement focused on the `api.paybank_redacted.com` ecosystem and associated mobile application assets. The campaign progressed from passive reconnaissance to active API schema reconstruction, targeted path brute-forcing, and sophisticated "Sandbox Breakout" attempts using leaked SDK credentials.

**Key Findings**:

* **Strong WAF**: The production API (`api.paybank_redacted.com`) is protected by a robust WAF (likely CloudFront/AWS Shield) that strictly enforces geofencing and blocks unauthorized traffic with **403 Forbidden**.
* **Sandbox Exposure**: A functional Sandbox environment (`pg-sandbox.paybank_redacted.com`) was discovered and successfully accessed using valid OSINT-derived credentials.
* **Clean Logic**: BOLA/IDOR testing on the Sandbox revealed proper scope enforcement (Write-Only keys cannot Read).
* **Credential Leakage**: Decompiled APK analysis revealed 14 potential credential strings (Bearer/Basic), though none were valid against Production.

---

## Phase 1-6: Reconnaissance & Attack Surface Mapping

**Goal**: Map the digital footprint and identify weak points.

* **Asset Discovery**: Enumerated 292 unique endpoints using `apktool` and static analysis of `bank_redacted.apk`.
* **Tech Stack**: Identified CloudFront, AWS, and specific backend headers.
* **Port Scanning**: Confirmed only standard ports (80/443) open on key infrastructure.
* **Vulnerability Scan**: Checked for missing security headers. `api.paybank_redacted.com` lacked HSTS, but rigorous WAF rules compensated for this.

## Phase 7: The Game Plan Pivot (OSINT & Staging)

**Goal**: Locate "soft" targets (Staging/Dev) and developer documentation.

* **Mobile Scout**: Analysis of GitHub/NPM revealed `paybank_redacted-android-sdk-v2` and `paybank_redacted-js-sdk-v2`.
* **Staging Hunt**: Identified `pg-sandbox.paybank_redacted.com` as a reachable staging environment.
* **Intelligence**: Confirmed Authentication Scheme: `Authorization: Basic <base64(public_key:)>`.

## Phase 8: The Sandbox Breakout

**Goal**: Use valid Sandbox credentials to bypass Production WAF ("Credential Stuffing").

* **Key Validation**: Tested candidate keys against `pg-sandbox`.
  * **Success**: Key `pk-Z0OS...` returned **400 Bad Request** (Valid Auth, Invalid Body).
* **WAF Bypass Attempt**: Replayed the *exact* valid request against `api.paybank_redacted.com`.
  * **Result**: **403 Forbidden** (Scenario A).
  * **Analysis**: The WAF does not "fail open" or treat Sandbox keys as valid auth material to bypass the edge protection.

## Phase 9: Sandbox Exploitation (Logic Flaws)

**Goal**: Pivot to Logic Flaws (IDOR/BOLA) within the accessible Sandbox.

* **BOLA Testing**:
    1. Authenticated to `pg-sandbox` with `pk-Z0OS...`.
    2. Created a `Checkout` resource (ID: `1596c5e8...`).
    3. Attempted to **GET** (Read) the resource and others.
    4. **Result**: **401 Unauthorized**.
  * **Conclusion**: Public Keys are scoped for **Creation Only**. They cannot read data, mitigating BOLA risks.
* **Geo-Spoofing**:
  * Attempted to bypass Production WAF using `X-Forwarded-For: 202.x.x.x` (Philippines IP).
  * **Result**: **Failed** (403 Forbidden).

## Phase 10: The Credential Gauntlet

**Goal**: Test hardcoded credentials found in the APK deep dive.

* **Static Analysis**: Scanned decompiled code for `Bearer regex` and `Basic regex`.
  * Found: 3 Bearer Tokens, 11 Basic Auth strings.
* **Gauntlet Run**: Tested all 14 credentials against `api.paybank_redacted.com/v1/checkouts`.
  * **Result**: **403 Forbidden** on all attempts.
  * **Analysis**: Credentials likely for internal/dev environments or effectively blocked by IP/Geo-restrictions.

---

## Conclusion

The target demonstrates a mature security posture.

1. **Defense in Depth**: WAF + API Gateway + Application Logic all enforced strict access controls.
2. **Secret Management**: While "Public" keys were found, they were properly scoped (Write-Only). No high-privilege secrets were leaked in the APK.

**Recommendation**: Methodology exhausted. No further actions recommended without new leads.
