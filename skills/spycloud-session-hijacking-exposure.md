---
name: Find stolen session cookies for a domain
description: Detect session-hijacking exposure by retrieving session cookies recaptured from infostealer malware
  for a given cookie domain, so live sessions can be invalidated.
api: openapi/spycloud-session-identity-protection-openapi.yml
operations:
- sip-get-cookies-for-domain
- sip-list-all-breach-metadata
- sip-get-metadata-for-a-breach
---

# Find stolen session cookies for a domain

## Goal

Find out whether valid session cookies for your domain are circulating, and which ones to kill.

## Steps

1. **Query the domain.** `GET /breach/data/cookie-domains/{cookie_domain}` (`sip-get-cookies-for-domain`).
   Results include **all subdomains** of the domain given; pass a specific subdomain to narrow.
2. **Filter to what matters.** `cookie_name` is **case sensitive** — use the exact name of your session
   cookie. `since_cookie_expiration` / `until_cookie_expiration` bound by expiry, which is how you separate
   cookies that are still usable from ones that already lapsed.
3. **Page.** Follow `cursor` to exhaustion; the 2-minute cursor TTL applies.
4. **Attribute the source.** `GET /breach/catalog/{id}` (`sip-get-metadata-for-a-breach`) or
   `GET /breach/catalog` (`sip-list-all-breach-metadata`) resolves `source_id` to the malware corpus.

## Acting on it

- A cookie whose expiration is in the future is a **live** hijacking risk. Invalidate the server-side session,
  not just the cookie, and force re-auth.
- Password rotation does not revoke an existing session in most stacks. If you only reset the password, the
  stolen cookie keeps working — that is the whole reason this endpoint exists.
- A cookie hit almost always implies an infected device. Pivot to the Compass malware skill using the same
  identity, and remediate the endpoint.

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
