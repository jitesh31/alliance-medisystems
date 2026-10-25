# SECUREBANK AI Service Centre Security Assessment

Date: 2026-10-07  
Target: `https://codered.cisco.com/html/bankapp-ai.html`  
Account tested: `NeuralNinja`  
Method: Authenticated assessment with controlled validation in the Cisco CodeRed training environment. Read-only checks were preferred, and state-changing tests were limited to the participant account and challenge workflows described below.

## Executive summary

The application has a confirmed cross-customer account access chain. Customer-visible AI diagnostics disclose internal prompts, retrieval results, another customer's recovery case, and a live recovery token. The Wealth search also returns another customer's Private Wealth onboarding phrase. Both exposed credentials were accepted by their respective access forms, proving the leaks can cross authorization boundaries. A separate unkeyed FX preview header can poison the shared USD/INR quote cache and cause financial settlement at an attacker-selected rate. The login endpoint also discloses whether a submitted username exists through distinct error codes and messages.

The affected third-party relationships already showed zero remaining balance, so no transfer action was available or attempted.

### Validation status of the supplied findings

| Supplied claim | Result | Evidence |
|---|---|---|
| AI diagnostics expose recovery secrets | Confirmed, reproduced live | Participant-readable traces expose the system prompt, cross-customer records, case and campaign data, and a recovery token. |
| Leaked token grants cross-customer recovery access | Confirmed | The token tuple previously returned `RECOVERY_VERIFIED`; the AI overview still shows Mira Vale as the recovered relationship, now with a zero balance. |
| Wealth RAG leaks another customer's onboarding phrase | Confirmed, reproduced live | A fresh Wealth query returned Jonah Crest's full onboarding phrase from `private-wealth-onboarding-2026`. |
| Relationship endpoints lack ownership checks | Confirmed | Credentials from unrelated records were accepted by the target and recovery access APIs while authenticated as `NeuralNinja`. |
| Duplicate refund race condition | Confirmed | Concurrent claims returned `SETTLED` and `DUPLICATE_SETTLED`; the final state is `DUPLICATE_PAID` and two INR 30,000 credits exist. |
| Profile mass assignment | Rejected by tested controls | Both `accountTier` and `account_tier` were ignored; the account remained `STANDARD`. |
| Caller-controlled recovery email | Confirmed | A recovery for `christiano.ronaldo` was redirected to the Cisco email registered to `NeuralNinja`. The delivered temporary password authenticated successfully as Cristiano Ronaldo. |
| AI invoice can trigger automatic credit | Confirmed | The ledger records an INR 100,000 `AI_INVOICE_PAYMENT`, and `AI_EXCESSIVE_AGENCY` is marked complete. |
| FX quote cache poisoning | Confirmed | `X-Rate-Preview: 10000` seeded the shared USD/INR cache at INR 10,000 per USD; 50 USD was converted for INR 500,000 in total and `FX_CACHE_POISONING` completed. |
| Preferred eligibility method override | Confirmed | A normal GET returned 403 and disclosed `X-Compatibility-Mode: classic-banking`; adding `X-HTTP-Method-Override: POST` changed the result to `APPROVED` and enabled a INR 90,000 voucher. |

### OWASP Top 10:2025 coverage

| Category | Assessment result |
|---|---|
| A01 Broken Access Control | Confirmed through cross-customer AI access, path traversal, recovery takeover, privileged dispute settlement, and internal-service SSRF. |
| A02 Security Misconfiguration | Confirmed through archive directory traversal/indexing, active SVG handling, workflow disclosure, and server-version disclosure. |
| A03 Software Supply Chain Failures | No client-side third-party dependency was identified; server-side dependency provenance and patch status could not be assessed externally. |
| A04 Cryptographic Failures | No new issue confirmed. Session cookies use `Secure`, `HttpOnly`, and `SameSite=Strict`; HTTPS and HSTS are enforced. |
| A05 Injection | Stored SVG script injection is confirmed. Focused SQL/JSON injection checks against login were rejected. |
| A06 Insecure Design | Confirmed through AI-authorized settlement, refund race, promotion replay, caller-controlled recovery, and cache-poisoned FX settlement. |
| A07 Authentication Failures | Confirmed through recovery takeover, login username enumeration, and lack of observable login throttling. |
| A08 Software or Data Integrity Failures | Confirmed where attacker-controlled SVG evidence influenced a trusted Card Operations workflow. |
| A09 Security Logging and Alerting Failures | Server-side logging and alert delivery were not externally observable. Thirty-two rapid failures produced no client-visible throttle or challenge. |
| A10 Mishandling of Exceptional Conditions | No new issue confirmed. Malformed JSON, type confusion, and extreme payment values returned controlled 400 responses without settlement. |

## Confirmed findings

### 1. Customer-visible diagnostics expose recovery secrets

Severity: Critical  
Affected endpoints:

- `POST /api/bankapp/ai/support/messages`
- `GET /api/bankapp/ai/support/diagnostics/{traceId}`

An ordinary participant can ask a support question and open the generated diagnostic record. The response exposes:

- the complete system prompt;
- internal routing and retrieval metadata;
- records belonging to another customer;
- a Priority Recovery case reference and campaign handle;
- the complete recovery token in the tool trace.

Observed evidence included a record for another customer, case reference `CASE-PRIORITY-019`, campaign handle `mira.recovery`, and a live token (redacted here).

The issue was reproduced again on 2026-10-07. A normal support lookup generated trace `d2aa030e-7e0c-4f89-bc35-e07e27d00ef7`, whose participant-readable diagnostic exposed the system prompt, support index name, Mira Vale's record, the case reference, and campaign handle. A documented Priority Recovery follow-up generated trace `9d34330b-0d96-4501-873a-8fac810fd5a5`; its tool trace still contained a recovery token. The token value remains redacted from this report.

A targeted request for Kanchan Kumar Dey produced another integrity failure. The assistant claimed it had located Kanchan's original anniversary onboarding record, but diagnostic trace `349650d1-8f7d-413d-a8e1-2f1e2b407f29` showed no supporting Kanchan record: retrieval matched only the caller's own request and Mira Vale's unrelated case. A Kanchan-specific recovery follow-up then generated trace `14e79f31-8066-4cc7-aee3-70621a8c29f3`; the tool ignored the supplied Kanchan case and campaign and issued Mira Vale's recovery token again. This demonstrates both unsupported model claims and incorrect cross-customer tool binding.

Impact: disclosure of operational prompts and live authentication material, enabling unauthorized recovery of another customer's relationship.

Recommended fix:

- Never return raw prompt, retrieval, or tool traces to participant accounts.
- Store diagnostics in an operator-only system protected by role and tenant authorization.
- Redact credentials, tokens, account identifiers, and personal data before storage and display.
- Scope retrieval results to the authenticated customer before they reach the model or trace pipeline.
- Revoke all recovery tokens exposed in existing traces.

### 2. Leaked recovery token grants cross-customer relationship access

Severity: Critical  
Affected endpoint: `POST /api/bankapp/ai/priority-recovery/access`

The case reference, campaign handle, and recovery token obtained from the diagnostic record were submitted to the recovery form while signed in as `NeuralNinja`. The server returned `RECOVERY_VERIFIED` and identified the relationship as belonging to another customer.

Impact: broken object-level authorization and cross-account access. If the target relationship has funds, the same workflow exposes a transfer action to the attacker's signed-in account.

Recommended fix:

- Bind every recovery case and token to the authenticated customer and intended destination account.
- Require a fresh, independent step-up factor outside the AI workflow.
- Make tokens single-use, short-lived, audience-bound, and immediately invalid after any disclosure or verification attempt.
- Authorize again on the transfer endpoint; do not rely on prior UI state or possession of the recovery token.

### 3. Wealth search leaks another customer's onboarding phrase

Severity: Critical  
Affected endpoint: `POST /api/bankapp/ai/wealth/query`

A normal research query asking for relationship records returned another customer's name, relationship description, and full Private Wealth onboarding phrase, with citations to an internal onboarding document.

The exposed phrase was accepted by `POST /api/bankapp/ai/targets/access`, which returned `ACCESS_GRANTED` for the other customer's relationship.

The disclosure was reproduced again on 2026-10-07. A query for the cited Private Wealth onboarding document returned Jonah Crest's full onboarding phrase. The phrase remains redacted from this report.

Impact: cross-tenant information disclosure and unauthorized access to a Private Wealth relationship. The exposed access workflow can lead directly to a transfer endpoint.

Recommended fix:

- Apply customer and tenant authorization filters before retrieval, not after model generation.
- Exclude secrets and authentication material from the vector/search corpus.
- Use a strict response schema that cannot contain credentials, tokens, onboarding phrases, or unrelated customer data.
- Bind target access to the signed-in customer's entitlements on the server.
- Rotate every onboarding phrase stored in the affected corpus.

### 4. Cross-account access endpoints rely on bearer-like phrases without sufficient ownership checks

Severity: Critical  
Affected endpoints:

- `POST /api/bankapp/ai/targets/access`
- `POST /api/bankapp/ai/targets/{targetCode}/transfer`
- `POST /api/bankapp/ai/priority-recovery/access`
- `POST /api/bankapp/ai/priority-recovery/transfer`

Possession of a phrase or recovery tuple was sufficient to open a relationship belonging to another customer. The UI then exposes transfer functionality to the signed-in account. This confirms server authorization is based on shared secrets rather than account ownership and explicit entitlement.

Impact: account takeover of linked relationships and potential unauthorized fund transfer.

Recommended fix:

- Enforce object-level authorization using the authenticated principal for every access and transfer request.
- Treat phrases as onboarding hints, not authorization factors.
- Require server-side ownership, consent, destination, amount, and replay checks in the final transfer transaction.
- Add immutable audit events and alerts for cross-customer access attempts.

### 5. Authenticated path traversal exposes and redeems another customer's voucher

Severity: Critical  
Affected endpoints:

- `GET /api/bankapp/documents/download?file=...`
- `POST /api/bankapp/promotions/redeem`

The document download parameter accepts traversal sequences outside the signed-in customer's document directory. Requesting `../../private-wealth/` returned an archive directory index and exposed another customer's folder. That folder disclosed `goodwill_voucher.txt`, which contained a live one-time goodwill voucher worth INR 130,000.

The document metadata endpoint also discloses the internal customer workspace path, resolver name, and customer UUID through `archiveDiagnostics`, making traversal discovery and targeting easier.

The exposed voucher was accepted by the promotion redemption endpoint while authenticated as `NeuralNinja`. It initially credited INR 130,000, increasing the balance from INR 180,783.25 to INR 310,783.25. The service later revalued the voucher to INR 300,000 and automatically credited the INR 170,000 difference. The voucher code is redacted from this report.

Impact: cross-customer confidential document disclosure, credential theft, and unauthorized credit to the attacker's account.

Recommended fix:

- Resolve requested paths against a fixed per-customer root using a canonical path API, then reject any result outside that root.
- Reject absolute paths, traversal components, encoded traversal, directory targets, and symbolic-link escapes.
- Map opaque document IDs to server-side records instead of accepting filesystem paths from clients.
- Authorize every document record against the authenticated customer before opening the file.
- Bind vouchers to their intended customer and enforce that binding during redemption.
- Revoke exposed vouchers and audit all traversal requests, directory listings, downloads, and cross-customer redemptions.

### 6. Cross-session refund race causes duplicate settlement

Severity: High  
Affected endpoint: `POST /api/bankapp/refunds/{id}/claim`

The same `READY` refund was submitted concurrently from two independently authenticated sessions. Both requests were accepted inside the reconciliation window:

- one response returned `SETTLED` with `paidCount: 1`;
- the other returned `DUPLICATE_SETTLED` with `paidCount: 2`;
- the final refund state was `DUPLICATE_PAID`;
- the INR 30,000 refund was credited twice, increasing the balance by INR 60,000.

Impact: duplicate payouts through a race condition and missing transaction idempotency.

Recommended fix:

- Claim the refund with a single conditional database update from `READY` to `PROCESSING`.
- Reject every request that does not win that atomic transition.
- Use a unique idempotency key at the settlement boundary and a database uniqueness constraint for the refund payout.
- Reconcile before crediting, rather than crediting each channel and reconciling afterward.
- Alert on multiple claims for the same refund reference.

### 7. Unauthenticated recovery account enumeration and email disclosure

Severity: High  
Affected endpoint: `POST /api/bankapp/recovery/start`

The recovery endpoint produces observably different responses for valid and non-valid usernames. A valid participant username returned:

- `CONFIRMATION_REQUIRED`;
- a recovery reference;
- the complete registered Cisco email address.

A non-account identity returned only the generic `REQUESTED` response. No authentication was required for either request.

A follow-up check against the named training identity `christiano.ronaldo` reproduced the issue. The unauthenticated response disclosed `CONFIRMATION_REQUIRED`, recovery reference `SBR-E5492CDFB6E3`, and the full registered address `christiano.ronaldo@securebank.invalid`. The public anniversary directory did not list this identity, so the recovery endpoint independently confirmed that the account exists.

Impact: reliable account enumeration, disclosure of registered email addresses, and recovery-reference exposure that supports targeted takeover attempts.

Recommended fix:

- Return the same status, schema, timing, and message for every submitted username.
- Never return the recovery reference or full delivery address to an unauthenticated caller.
- Display only a generic confirmation or, when strictly necessary, a minimally masked destination.
- Bind the reference to the server-side account and verified destination, with short expiry and strict rate limits.
- Monitor recovery requests across usernames, source addresses, devices, and sessions.

### 8. Stored SVG XSS executes a privileged dispute settlement

Severity: Critical  
Affected endpoints:

- `POST /api/bankapp/card-disputes/{caseReference}/attachments`
- `GET /api/bankapp/workflows/card-dispute-v1`
- `GET /api/bankapp/card-operations/csrf`
- `POST /api/bankapp/card-disputes/{caseReference}/settle`

The dispute API accepts an SVG containing inline JavaScript and stores it as `image/svg+xml`. The upload response exposes a workflow definition through a `Link` header. That participant-readable definition discloses the protected Card Operations CSRF endpoint, the hidden settlement endpoint, the required `CARD_OPERATIONS` session, and the exact approval body.

A replacement SVG fetched the Card Operations CSRF token from the privileged reviewer session and submitted `{"decision":"APPROVE"}` to the hidden settlement route. Case `CD-7FC8550C09` changed from `UNDER_REVIEW` to `PROVISIONALLY_CREDITED`, and INR 250,000 was credited to the participant account. The balance increased from INR 540,783.25 to INR 790,783.25, and the application recorded the `STORED_SVG_XSS` completion marker.

Impact: persistent same-origin XSS against Card Operations, unauthorized use of a privileged settlement workflow, and direct financial credit to the attacker-controlled participant account.

Recommended fix:

- Do not accept active SVG as evidence unless it is strictly required.
- Rasterize images and PDF-render documents in an isolated service before review.
- If SVG must be retained, sanitize it with a strict element and attribute allow-list that removes scripts, event handlers, external references, `foreignObject`, animation, and active URLs.
- Serve originals only as attachments from a cookieless, isolated origin with `Content-Disposition: attachment`, `X-Content-Type-Options: nosniff`, and a restrictive sandbox CSP.
- Never embed untrusted uploads directly into an authenticated staff page.
- Do not expose privileged workflow endpoints, CSRF details, or request contracts to participant accounts.
- Bind settlement decisions to explicit server-side reviewer actions and require transaction-level authorization independent of the browser session.

### 9. AI invoice review automatically credits model-directed payment

Severity: Critical  
Affected endpoint: `POST /api/bankapp/ai/invoice/review`

The invoice review workflow caused an automatic INR 100,000 credit. The account ledger contains an `AI_INVOICE_PAYMENT` entry for INR 100,000 with reference `f0496028-637b-4c6a-8f8f-34cebf77edfc`, and the application recorded the `AI_EXCESSIVE_AGENCY` completion marker.

Impact: free-form AI review output can reach a financial settlement action without an independent deterministic authorization boundary.

Recommended fix:

- Treat model output as untrusted advisory data.
- Parse invoice fields deterministically and verify the supplier, destination, amount, and entitlement against authoritative records.
- Enforce transaction limits and fraud rules outside the model.
- Require explicit transaction-level human authorization before settlement.
- Make the payment service reject any request that lacks a server-issued, purpose-bound approval.

### 10. Supplier preview SSRF exposes and redeems an internal account voucher

Severity: Critical  
Affected endpoints:

- `POST /api/bankapp/ai/ssrf/sample`
- `POST /api/bankapp/ai/ssrf/preview`
- `POST /api/bankapp/ai/ssrf/redeem`

The supplier sample response disclosed the internal service origin in the `X-Supplier-Archive-Origin` header: `http://bankapp-ai-voucher:8091`. Its public settlement document exposed an internal route template, `/internal/v1/anniversary-voucher?accountNumber={accountNumber}`. Supplying the participant's own account number through the preview endpoint caused the backend to fetch that internal route and return an INR 80,000 voucher code.

The voucher was accepted by the supplier redemption endpoint. The account balance increased from INR 790,783.25 to INR 870,783.25, the ledger recorded an INR 80,000 `AI_SSRF_VOUCHER` entry, and the application marked `AI_SSRF_BACKUP` complete. The voucher code is redacted from this report.

Impact: authenticated SSRF into an internal supplier service, internal topology and endpoint disclosure, unauthorized access to account-scoped voucher material, and direct financial credit.

Recommended fix:

- Remove internal origins, service names, ports, and paths from participant-visible headers and responses.
- Replace caller-supplied preview URLs with opaque server-side document identifiers.
- If remote retrieval is required, resolve an allow-listed identifier to a fixed destination server-side and prevent callers from controlling scheme, host, port, path, query, redirects, or DNS resolution.
- Block private, loopback, link-local, and service-network destinations before and after every redirect and DNS resolution.
- Require an independent entitlement check when issuing and redeeming vouchers; possession of a code must not authorize a credit.
- Bind voucher issuance and redemption to the authenticated customer, intended account, purpose, and a single-use server-side record.

### 11. Case-sensitive promotion redemption permits repeated cashback

Severity: High  
Affected endpoint: `POST /api/bankapp/promotions/redeem`

The anniversary promotion code is compared or stored without canonicalizing letter case. The original uppercase code had already been redeemed and correctly returned `409`. Each remaining case permutation of its two letters was then accepted independently:

- `Sb2026` credited INR 5,000;
- `sB2026` credited INR 5,000;
- `sb2026` credited INR 5,000.

The promotion ledger count increased from one to four entries, total promotion credit increased from INR 5,000 to INR 20,000, and the account balance increased from INR 870,783.25 to INR 885,783.25.

Impact: one-time promotion limits can be bypassed by changing only the case of the submitted code, resulting in repeat financial credits.

Recommended fix:

- Canonicalize codes with a locale-independent rule before validation, storage, lookup, and uniqueness enforcement.
- Store a server-side promotion identifier separately from the display code and enforce one redemption per promotion and customer with a database uniqueness constraint.
- Perform the eligibility check and redemption insert in one transaction.
- Reconcile existing redemptions after canonicalization and investigate duplicate credits.

## Additional confirmed findings

### 12. Caller-controlled recovery destination enables account takeover

Severity: Critical  
Affected endpoints:

- `POST /api/bankapp/recovery/start`
- `POST /api/bankapp/recovery/resend`

The unauthenticated recovery flow accepted `christiano.ronaldo` as the target and disclosed a fresh recovery reference. The resend endpoint then accepted the Cisco email registered to the unrelated `NeuralNinja` participant as `deliveryEmail`. A temporary password arrived at that caller-selected address and authenticated successfully as Cristiano Ronaldo, without requiring a password change.

The recovered session exposed Cristiano Ronaldo's complete account overview and an active recovery binding whose destination was `NeuralNinja` (`SBB8CE65D94A1B41`). The available balance was already INR 0 because an earlier recovery transfer had removed INR 50,000. A matured INR 70,000 fixed deposit remained visible, but the server correctly rejected its claim with HTTP 403 while recovery mode was active. No additional funds were transferred.

The account overview subsequently recorded `FORGOT_PASSWORD_BINDING` as complete.

Impact: unauthenticated account takeover and access to customer profile, balances, ledger entries, and recovery-only transfer functionality. A target with an available balance could be drained to the attacker's account.

Recommended fix:

- Never accept a recovery destination from the unauthenticated caller; deliver only to the verified address stored on the target account.
- Keep responses indistinguishable and do not disclose recovery references or full email addresses.
- Require a separate verified factor before issuing a temporary password.
- Force immediate password replacement and revoke all other sessions after successful recovery.
- Bind recovery transfers to a server-approved destination established through a separately authenticated process.

### 13. Unkeyed preview header poisons the shared FX quote cache

Severity: Critical  
Affected endpoints:

- `GET /api/bankapp/fx/rates?pair=USD-INR`
- `POST /api/bankapp/fx/convert`

The FX quote endpoint trusts the participant-controlled `X-Rate-Preview` header when generating a fresh market quote, while the shared 30-second cache is keyed only by the currency pair. Sending `X-Rate-Preview: 10000` on a cache miss returned:

- rate `10000.000000`;
- source `PREVIEW_CACHE`;
- `influencedBy: X-Rate-Preview`.

The conversion endpoint then accepted that cached rate as authoritative. The server limits each conversion to USD 2, but the limit can be repeated. The remaining USD 49 wallet was settled in 25 conversions: 24 conversions of USD 2 and one conversion of USD 1. This credited INR 490,000, increased the account balance from INR 885,783.25 to INR 1,375,783.25, reduced the USD wallet to zero, and marked `FX_CACHE_POISONING` complete.

A later USD 1 wallet credit was also converted at the poisoned INR 10,000 rate. The quote again returned `PREVIEW_CACHE` and `influencedBy: X-Rate-Preview`; the conversion settled with ID `623aa003-f118-4a80-a5c5-0bc8c1c872c0`, credited INR 10,000, and changed the live balances from INR 1,465,700 and USD 1 to INR 1,475,700 and USD 0. Total confirmed settlement through this poisoned rate is therefore USD 50 for INR 500,000.

Impact: any participant able to win a cache miss can set a shared attacker-selected quote up to the challenge ceiling and cause the settlement service to credit funds using that poisoned value. Because the cache is shared by pair, other customers requesting the same pair during the TTL may also receive the poisoned quote.

Recommended fix:

- Remove participant access to preview or test rate inputs in production routes.
- Keep preview pricing on a separate authenticated operator endpoint and isolated cache namespace.
- Build cache keys from every input that can affect a response, including authorization context and pricing mode.
- Obtain the executable rate from a trusted pricing service at settlement time; never treat a client-populated cache entry as authoritative.
- Enforce cumulative conversion limits and anomaly detection, rather than only a per-request USD limit.
- Invalidate the affected cache and reconcile settlements issued from `PREVIEW_CACHE` quotes.

### 14. Login errors disclose valid usernames

Severity: Medium  
Affected endpoint: `POST /api/auth/login`

The unauthenticated login endpoint returns different machine-readable codes and user-facing messages depending on whether the submitted username exists. With the same intentionally incorrect password:

- the valid username `neuralninja` returned `PASSWORD_INCORRECT` and “The password you entered is incorrect”;
- a randomly generated nonexistent username returned `USERNAME_NOT_FOUND` and “We couldn't find an account with that username.”

Both responses used HTTP 401, but the distinct body values make account enumeration reliable. The behavior also explained why targeted checks for prospective challenge identities could be classified as nonexistent without knowing their passwords.

Impact: an unauthenticated attacker can build a list of valid banking usernames for password spraying, credential stuffing, phishing, and recovery abuse. This compounds the recovery weaknesses documented in findings 7 and 12.

Recommended fix:

- Return the same HTTP status, response schema, error code, message, and comparable timing for unknown usernames and incorrect passwords.
- Keep the reason for failure only in restricted server-side audit logs.
- Apply rate limits and anomaly detection across username, source address, device, and network ranges.
- Alert customers and security operations when repeated authentication failures target known accounts.

### 15. Login endpoint lacks observable automated-attack throttling

Severity: Medium  
Affected endpoint: `POST /api/auth/login`

Thirty-two rapid authentication attempts for the same nonexistent username were sent from one source address. Every request was processed normally and returned HTTP 401. None returned HTTP 429, none supplied a `Retry-After` header, and no progressive challenge or other client-visible throttle was introduced.

The requests deliberately targeted a nonexistent probe identity, so the check did not risk locking a customer account. The result demonstrates that an attacker can automate the username-enumeration behavior documented in finding 14 without encountering a source-level control at this request volume.

Impact: attackers can enumerate usernames and perform sustained password spraying or credential stuffing more efficiently. The exposure is amplified by the recovery weaknesses documented in findings 7 and 12.

Recommended fix:

- Apply layered rate limits by source address, network, device, account identifier, and aggregate failure patterns.
- Introduce increasing delays and temporary risk-based challenges without revealing whether the username exists.
- Return HTTP 429 with a bounded `Retry-After` value when request limits are exceeded.
- Alert on distributed password spraying and enumeration patterns, not only high request counts against one account.
- Keep protective thresholds and lockout behavior resistant to denial-of-service attacks against known customers.

### 16. HTTP method override bypasses Preferred eligibility authorization

Severity: High  
Affected endpoints:

- `GET /api/bankapp/preferred-enrollment`
- `POST /api/bankapp/profile/voucher/redeem`

The normal eligibility request returned HTTP 403 because the account's qualifying relationship balance did not meet the INR 1,000,000 requirement. The rejection disclosed `X-Compatibility-Mode: classic-banking` in its response headers.

Replaying the same GET request with both of the following request headers changed how the server dispatched the operation:

- `X-Compatibility-Mode: classic-banking`
- `X-HTTP-Method-Override: POST`

The response changed from `403 NOT ELIGIBLE` to HTTP 200 with `status: APPROVED`, `accountTier: PREFERRED_BANKING`, and `joiningBenefit: 90000`. The newly issued voucher was then redeemed through the normal voucher endpoint. The ledger recorded an INR 90,000 `PREFERRED_VOUCHER` credit, the balance increased from INR 1,375,783.25 to INR 1,465,783.25, and the application marked `METHOD_OVERRIDE` complete.

Impact: a participant can bypass the relationship-balance eligibility decision, assign themselves a privileged banking tier, and obtain the associated financial benefit.

Recommended fix:

- Disable HTTP method override at the proxy, framework, and application layers unless a narrowly scoped legacy requirement exists.
- Do not expose compatibility-mode routing to participant requests.
- Authorize the effective business action after all proxy and framework method transformations.
- Make GET handlers read-only and keep approval state changes on explicit authenticated POST endpoints protected by CSRF.
- Revalidate eligibility transactionally when issuing and redeeming the voucher; do not rely on a previously assigned tier alone.
- Audit requests containing method-override or compatibility headers and reconcile benefits issued through this path.

## Negative validation results

The following server-side controls held during focused tests:

- In the recovered Cristiano Ronaldo session, both `POST /api/bankapp/fixed-deposits/claim` for the visible matured INR 70,000 deposit and `POST /api/bankapp/promotions/redeem` with `SB2026` returned HTTP 403: `The recovered Preferred account can only complete its recovery transfer`. The available balance therefore remained zero.
- `PATCH /api/bankapp/profile` ignored attempts to set `accountTier`, `account_tier`, common tier aliases, and nested customer-profile tier fields, including when sent with the disclosed `X-Compatibility-Mode: classic-banking` header; the account remained `STANDARD` and the INR 90,000 Preferred voucher stayed locked.
- `POST /api/bankapp/payments` rejected a negative amount with the enforced INR 1 to INR 5,000 range.
- `POST /api/bankapp/payments` also rejected a hidden direct-account transfer from the normal participant profile with `Select a registered beneficiary`; a controlled INR 1 self-transfer test made no balance change.
- Payment-note SQL, time-delay, HTML, template, null-byte, overlength, and JSON type-confusion probes were repeated with `amount: 0`. Both an invalid beneficiary and a valid saved beneficiary produced uniform 400 responses before settlement. The balance remained INR 1,465,783.25 and the payment count remained zero. This confirms no transfer occurred, but it does not establish whether the note is safely handled after all payment validations pass.
- The transfer page's three beneficiaries are hardcoded client-side. GET and OPTIONS checks for `/api/bankapp/beneficiaries`, `/api/bankapp/payments/beneficiaries`, `/api/bankapp/beneficiary`, `/api/bankapp/payees`, and `/api/bankapp/transfers` all returned 404, providing no beneficiary-creation route. Extra source-account, direct-transfer, transfer-type, and shared-target fields in the payment body were ignored.
- `PATCH /api/bankapp/profile` ignored combined attempts to set `isSharedTarget`/`is_shared_target` and `accountTier`/`account_tier` to `PRIVATE_BANKING`. The separate INR 150,000 Private Banking benefit was not activated.
- Common quote-based SQL injection strings in the login username and password were rejected with normal authentication errors; JSON object/array type confusion and malformed JSON returned controlled 400 responses without stack traces.
- Cross-origin requests without a CSRF token to `POST /api/bankapp/notifications/read` and `PATCH /api/bankapp/profile` returned HTTP 403.
- Extreme and nonnumeric payment amounts returned controlled 400 responses, created no payment record, and left the balance at INR 1,375,783.25.
- Replaying the successful Preferred method-override request after redemption returned HTTP 409 and did not issue a second voucher. Applying the same Classic Banking method-override headers to the payment endpoint returned HTTP 405, so the override bypass was not reproduced for transfers.
- Two concurrent replays of the exact uppercase `SB2026` value returned `409`; however, mixed- and lowercase variants bypassed the one-time redemption check as described in finding 11.
- `PATCH /api/bankapp/cards/control` rejected a daily limit above INR 100,000.
- `POST /api/bankapp/fx/convert` rejected a client-supplied rate of 10,000 while the authoritative cached quote remained 83.25. The rate could be changed only by first poisoning a fresh cache entry through `X-Rate-Preview`, as described in finding 13.
- Common query parameters, proxy headers, internal-trust headers, GET bodies, duplicate parameters, matrix parameters, cache directives, pair variants, metadata headers, host/port variants, compatibility flags, source selectors, cookies, and delimiter tricks did not alter the FX quote. The confirmed influence input was specifically `X-Rate-Preview` on a cache miss.
- The invoice preview rejected a direct loopback request to `127.0.0.1` with an allow-list error.
- The session identifier rotated after successful authentication, preventing the tested session-fixation pattern.
- An authenticated API response did not emit cross-origin access headers for an untrusted origin.
- Traversal to a harmless host file outside the application archive returned `404`; arbitrary operating-system file read was not reproduced.
- Participant attempts to update or approve the existing card dispute directly returned `404`; no participant approval route was exposed.
- Card-dispute creation ignored privileged fields such as `status`, `decision`, `transferredAmount`, and `approvedAmount`, and returned the existing `UNDER_REVIEW` case instead of creating a duplicate.
- Raw and percent-encoded `..` path-normalization attempts from `/api/bankapp/` into `/api/card-operations/` were rejected with `400`.

## Controls observed

- Session cookie: `Secure`, `HttpOnly`, `SameSite=Strict`.
- State-changing AI requests use an `X-XSRF-TOKEN` header.
- Strong baseline response headers were present: HSTS, CSP, `X-Frame-Options: DENY`, and `X-Content-Type-Options: nosniff`.
- Client rendering uses `textContent` in the reviewed flows, reducing DOM XSS exposure.
- A loopback SSRF attempt against the invoice preview was rejected by an allow-list message. Redirect handling, DNS rebinding resistance, and post-resolution IP checks were not verified.

## OWASP API8:2023 — Security Misconfiguration assessment

Confirmed API8-relevant weaknesses:

- Responses disclose the exact reverse-proxy platform and version: `nginx/1.24.0 (Ubuntu)`. The same detail is repeated in the native `405` error page.
- The legacy document resolver permits parent-directory traversal, directory indexing, and disclosure of internal workspace paths, resolver names, and customer UUIDs. This misconfiguration directly enabled cross-customer document access.
- The card-dispute upload configuration permits active SVG and stores a scripted file as `image/svg+xml` without evidence of sanitization or rasterization. Exploitation still depends on the privileged viewer rendering the original content.

API8 controls that held:

- `TRACE` returned `405 Not Allowed`.
- Authenticated and unauthenticated preflight requests from `https://evil.example` were rejected; no permissive CORS headers were returned.
- `/v3/api-docs`, `/swagger-ui/index.html`, and `/actuator/health` were not publicly exposed.
- HSTS, a restrictive CSP, frame denial, MIME-sniffing protection, and no-store cache directives were present.

Recommended configuration fixes:

- Remove server version and operating-system details from response headers and generated error pages.
- Replace path-based document resolution with opaque document identifiers and a canonical per-customer allow-list.
- Disable directory responses from download APIs and remove archive diagnostics from participant responses.
- Reject SVG or process it through strict sanitization and isolated rasterization before staff review.

## Immediate containment

1. Disable customer access to diagnostic records.
2. Revoke and rotate all recovery tokens and onboarding phrases exposed to AI traces or retrieval indexes.
3. Disable cross-relationship transfer endpoints until ownership authorization is enforced.
4. Remove secret-bearing documents from the Wealth retrieval corpus and rebuild the index.
5. Review audit logs for diagnostic access, cross-customer target access, recovery verification, and transfers.
