---
name: Screen a password against compromised-credential data
description: Check a proposed password at account creation or reset against SpyCloud recaptured credentials using
  a k-anonymity hash-prefix lookup, per NIST SP 800-63B.
api: openapi/spycloud-nist-password-openapi.yml
operations:
- nist-check-password-hash
---

# Screen a password against compromised-credential data

## Goal

Block a compromised password at signup or reset without transmitting the password.

## Steps

1. **Hash locally.** Compute the password hash with one of the supported algorithms and pass it as `type`:
   `ntlm`, `sha1`, `sha256`, `sha512`.
2. **Send the prefix only.** `GET /check/hashes/{hash}` (`nist-check-password-hash`) where `{hash}` is the
   **first 5 hexadecimal digits** of that hash, plus `type=<algorithm>`.
3. **Match locally.** Compare the returned hash set against your full hash. A match means the password is
   present in recaptured breach data.
4. **Act.** Reject the password and ask for another. Per NIST SP 800-63B this is a screening control at
   creation/change time, not a periodic expiry rule — do not use it to force rotation on a schedule.

## Rules

- Never send the full hash and never send the plaintext. The 5-character prefix is the whole point of the design.
- Choose the algorithm your credential store already uses so you are not hashing plaintext you did not otherwise need.
- This endpoint is a single operation with no pagination; a 400 means the prefix was not 5 hex digits or `type`
  was not one of the four supported values.

## Preconditions

- **Auth**: send `x-api-key: <key>` on every request. Keys come from the SpyCloud Customer Portal
  (https://portal.spycloud.com). The key must be called from an IP on the customer's allow-list or the
  request returns **403** — a 403 here means entitlement or source-IP, not a bad selector.
- **Transport**: HTTPS only (TLS 1.2/1.3). HTTP is rejected.
- **Entitlement**: each SpyCloud API is separately licensed. A key entitled to Investigations is not
  automatically entitled to Enterprise ATO. Treat 403 as "not licensed for this function".

## Rules the agent must follow

- **Pagination**: when a response carries a non-empty `cursor`, pass it back as `?cursor=` to get the next
  page. Pages are a fixed 1,000 records and the cursor **expires after ~2 minutes** — do not park a cursor
  and resume later, and do not parallelise page fetches across a long gap.
- **Selectors**: most selectors accept up to 10 comma-delimited values, and most accept a sha1/sha256/sha512
  hash instead of plaintext. **Prefer the hashed form** — this API returns recaptured breach data and there is
  no reason to transmit raw PII when a hash will do.
- **422 means "too broad"**: a query resolving to more than 2,000 potential matches fails. Narrow it —
  full email instead of a domain, full phone with country code.
- **429 means back off**: retry at 3s then 9s (exponential) or 3s then 4s (sliding), then log and fail open.
  Never hot-loop; 429 covers both the per-key rate limit and the monthly quota.
- **Dates**: `since` / `until` / `since_modification_date` / `until_modification_date` are ISO 8601 `yyyy-mm-dd`.
- **Masked fields**: a field returned as exactly eight asterisks is masked by the key's asset allow-list, not
  missing from the data. Do not report it as "no data".
- **No idempotency keys**: SpyCloud publishes no idempotency contract. The only write operations are on the
  Enterprise ATO watchlist — check with a read before retrying a create.

## Handling the data

This tool returns recaptured breach, malware and phishing data about real people, including credentials and
identity documents. Return only what the task needs, never echo a plaintext password or identity document
into a log or a chat transcript, and prefer counts and booleans over raw records when answering a yes/no
exposure question.

## Reference

- Guidelines: https://docs.spycloud.com/public-sc/docs/api-guidelines
- Data schema: https://docs.spycloud.com/public-sc/docs/data-schema
- Conventions: `conventions/spycloud-conventions.yml` · Errors: `errors/spycloud-problem-types.yml`
