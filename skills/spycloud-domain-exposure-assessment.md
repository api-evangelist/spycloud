---
name: Size up exposure for a domain or email without pulling records
description: Use the SpyCloud Prospecting API to get aggregate breach and malware exposure counts for a domain or
  email address over a lookback window.
api: openapi/spycloud-prospecting-openapi.yml
operations:
- get-stats-for-a-domain
- prospecting-get-stats-for-an-email
---

# Size up exposure for a domain or email without pulling records

## Goal

Get a number, not a record set — for scoping, prioritisation or a first conversation.

## Steps

1. **Domain.** `GET /stats/domains/{domain}` (`get-stats-for-a-domain`).
2. **Email.** `GET /stats/emails/{email}` (`prospecting-get-stats-for-an-email`). Pass `skip_domain=true` to
   return email stats only and skip the domain rollup — noticeably faster.
3. **Set `lookback`.** Accepts `30`, `90`, `180`, `365` or all time, counting back from now. Always state the
   window with the number; "1,400 exposures" is meaningless without it.

## Rules

- This API returns counts, not records. If someone needs the underlying identities, they need Enterprise ATO
  or Investigations entitlement — say that rather than trying to reconstruct records from stats.
- Counts are recaptured-exposure counts, not "accounts breached at this company". A record can come from a
  third-party breach or an infostealer log on a personal device. Do not present the number as a breach of the
  domain owner.

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
