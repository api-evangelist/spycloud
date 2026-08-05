---
name: Check a consumer login for credential exposure
description: Decide whether a consumer email, username, phone or IP is exposed in recaptured breach data at login
  or account creation, including a zero-knowledge credential check.
api: openapi/spycloud-consumer-ato-prevention-openapi.yml
operations:
- cap-zero-knowledge
- cap-get-records-by-email-address
- get-records-by-usernames
- cap-get-records-by-phone-number
- cap-get-records-by-ip-address
- cap-list-all-breach-metadata
---

# Check a consumer login for credential exposure

## Goal

Return a risk signal for one consumer identity in the login or signup path, without shipping their password anywhere.

## Steps

1. **Prefer the zero-knowledge path.** `GET /check/hashes/credentials/{hash_prefix}` (`cap-zero-knowledge`).
   The hash is built from email+password **or** username+password; you send only a 5-, 6-, 7- or 8-character
   prefix and match the returned set locally. This is the correct call in a live login flow — the full
   credential never leaves your system.
2. **Identity lookup when you need the detail.** `GET /breach/data/emails/{email}`
   (`cap-get-records-by-email-address`), `GET /breach/data/usernames/{username}` (`get-records-by-usernames`),
   `GET /breach/data/phone-numbers/{phone_number}` (`cap-get-records-by-phone-number`), or
   `GET /breach/data/ips/{ip}` (`cap-get-records-by-ip-address`, which also takes CIDR — use an underscore
   instead of the slash). All accept sha1/sha256/sha512 of the value; use it.
3. **Scope the window.** `since` / `until` on `spycloud_publish_date` keeps a login check fast and recent.
   For a step-up decision, "exposed in the last 90 days" is a very different answer from "exposed ever".
4. **Name the breaches only if asked.** `GET /breach/catalog/{id}` (`get-metadata-for-a-breach`) or
   `GET /breach/catalog/` (`cap-list-all-breach-metadata`).

## Decision guidance

- A zero-knowledge hit means *this exact credential pair* is recaptured — force a reset, do not just warn.
- An identity hit with no credential match means the person appears in breach data but this password is not
  known-exposed; step-up auth is proportionate, a forced reset usually is not.
- Answer with a boolean plus a count and a date range. Do not return the recaptured credentials themselves
  into a consumer-facing flow.

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
