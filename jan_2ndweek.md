**SSC Vulnerability Report Artifact**

# Broken Access Control: Client-Side Context Override in `getKeywordSuggestions`

**Vulnerability Type:** Business Logic Errors > Broken Access Control (IDOR)
**Severity:** P3 (Medium)
**Endpoint:** `https://www.target_example.com.au/graphql`

## 1. Executive Summary

The GraphQL endpoint `getKeywordSuggestions` relies on a client-controlled variable (`visitorId`) in the JSON body to determine the user context for personalization, rather than strictly deriving identity from the authenticated session (Cookies/Bearer Token).

This allows an authenticated attacker to "masquerade" as a different user (or an anonymous user) by injecting an arbitrary UUID into the `visitorId` variable. While the session remains authenticated, the business logic returns data (search history suggestions) associated with the injected ID, not the logged-in user. This is a "Confused Deputy" vulnerability where the API trusts user input over the source of truth (Auth0 Token).

## 2. Technical Analysis

The application uses a dual-source validation model that is inconsistent:

* **Infrastructure Layer (WAF/Gateway):** correctly validates the `Authorization: Bearer` token and Cookies.
* **Application Layer (GraphQL Resolver):** prioritizes the `variables.visitorId` from the POST body.

When a mismatch occurs (Auth Token = User A, Body Variable = User B), the application does not reject the request. Instead, it processes the logic using User B's context.

## 3. Steps to Reproduce

**Prerequisite:** A valid, logged-in account on `target_example.com.au`.

1. **Establish Baseline:** Log in and perform a unique search (e.g., "Cobol Mainframe") to populate your user search history.
2. **Verify Personalization:** Execute the standard `getKeywordSuggestions` query. Observe that "Cobol Mainframe" appears in the suggestions.
3. **The Attack:** Intercept the request and modify the `variables` JSON in the POST body. Change the `visitorId` to an arbitrary or null UUID (e.g., `11111111-1111-1111-1111-111111111111`), while keeping your valid Session Cookies intact.

**cURL Proof of Concept:**

```bash
curl 'https://www.target_example.com.au/graphql' \
  -H 'authority: www.target_example.com.au' \
  -H 'content-type: application/json' \
  -H 'authorization: Bearer [YOUR_VALID_TOKEN]' \
  -H 'cookie: [YOUR_VALID_COOKIES]' \
  --data-raw $'{"operationName":"getKeywordSuggestions","variables":{"country":"au","keyword":"Cobol","visitorId":"11111111-1111-1111-1111-111111111111"},"query":"query getKeywordSuggestions($keyword: String!, $country: CountryCodeIso2!, $visitorId: UUID) { searchKeywordsSuggest(query: $keyword, count: 10, countryCode2: $country, visitorId: $visitorId) { suggestions { ... on KeywordSuggestion { text } } } }"}'

```

4. **Observation:** The response returns **generic** suggestions (e.g., "cobol", "cics") instead of the user's personalized history ("Cobol Mainframe").
5. **Conclusion:** The server discarded the authenticated user's history in favor of the injected `visitorId`.

## 4. Impact Assessment

* **Information Leakage:** If an attacker can guess or enumerate valid UUIDs of other users, they can retrieve that victim's private search history suggestions.
* **Logic Manipulation:** An attacker can pollute the search history/metrics of other users by sending searches associated with their UUIDs.
* **Pattern Risk:** This demonstrates a systemic failure to enforce "Identity Source of Truth" in the GraphQL layer.

## 5. Negative Testing (Scope Limitation)

I attempted to escalate this vulnerability to PII endpoints to verify critical impact.

* **Test:** Attempted the same `visitorId` injection on the `GetPersonalDetails` (Profile) query.
* 
**Result:** The server **ignored** the injected variable and correctly returned the authenticated user's own email/name.


* **Conclusion:** The vulnerability appears isolated to the Search/Personalization logic and does not currently affect the Profile `viewer` resolver.

## 6. Recommendation

Ensure that user identity is strictly derived from the validated `Authorization` token or Session Cookie for *all* GraphQL resolvers. The `visitorId` variable in the request body should be ignored or validated to match the authenticated session ID.
