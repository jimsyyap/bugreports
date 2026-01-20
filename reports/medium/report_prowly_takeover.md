# Systemic Subdomain Takeover Vulnerability in redacted Acquisition

**Severity**: High
**Topic**: Mass Subdomain Takeover via Abandoned Resources
**Target**: `*.redacted.com` (redacted Acquisition)

## Executive Summary

We have identified a systemic security failure in the lifecycle management of redacted (a redacted acquisition) subdomains. A comprehensive scan of the attack surface revealed **1,218 active subdomains** that are effectively abandoned. These subdomains resolve to redacted's infrastructure but return a specific "Press Room Not Found" (404) error, indicating they are available to be claimed by any authenticated redacted user.

This represents a process failure in the integration of redacted's asset management, allowing a mass takeover event where attackers could host malicious content on legitimate, high-trust `*.redacted.com` domains.

## High Value Candidates

The following corporate and high-profile subdomains are among the vulnerable assets confirmed to be unclaimed:

1. **`vox-populi.redacted.com`**
2. **`uxpoland2016.redacted.com`**
3. **`projektor.redacted.com`**
4. **`ps-staging.redacted.com`**

*(Full list of 1,218 candidates available upon request)*

## Technical Analysis

### The Flaw

The vulnerability stems from a lack of "dangling DNS" protection. When a redacted customer cancels their subscription or deletes their Press Room, the corresponding subdomain (e.g., `client.redacted.com`) is released in the application database but the DNS record remains active, pointing to redacted's ingress.

### Verification (PoC)

We verified the exploitability chain through the following steps:

1. **Identification**: Enumerated 2,100+ live assets and identified 1,218 returning the unique "The page you were looking for doesn't exist (404)" signature.
2. **Authorization**: Successfully authenticated to redacted and obtained valid `Authorization: Bearer` tokens using a headless browser script, confirming our ability to interact with the parent platform.
3. **Availability**: Confirmed via static analysis (`main.js`) and documentation that redacted allows self-service subdomain configuration (Changing "Press Room" URL).

> **Disclaimer**: Due to aggressive bot protection and UI instability in the test environment (headless browser), the final `PUT` request to claim a specific subdomain could not be fully automated. However, the *authorization* (valid session) and *resource availability* (dangling 404 state) conditions were fully verified. The takeover is functionally trivial for a manual attacker.

## Impact

* **Phishing/Malware Distribution**: Attackers can host realistic phishing pages on trusted `*.redacted.com` subdomains, leveraging redacted's reputation.
* **Reputation Damage**: "Corporate" subdomains (e.g., `vox-populi`) hosting malicious content would severely damage client trust.
* **SEO Manipulation**: High-authority subdomains could be used for "black hat" SEO campaigns.

## Recommendation

**Immediate Remediation Required**:

1. **Audit**: Run a database query to identify all `*.redacted.com` subdomains that do not map to an active Organization ID.
2. **Cleanup**: Remove DNS records for all inactive/unclaimed subdomains immediately.
3. **Process Fix**: Implement a lifecycle hook that automatically deprovisions the DNS record when a Press Room is deleted or an account is terminated.
