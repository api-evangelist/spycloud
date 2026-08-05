---
name: Triage employee credential exposure
description: Find and triage recaptured credential exposures for a monitored workforce domain or employee, using
  the SpyCloud Enterprise ATO Prevention API watchlist.
api: openapi/spycloud-enterprise-ato-prevention-openapi.yml
operations:
- eap-list-all-identifiers
- eap-get-all-records-in-watchlist
- eap-get-records-by-domain
- eap-get-records-by-email-address
- eap-list-all-breach-metadata
- eap-get-metadata-for-a-breach
---

# Triage employee credential exposure

## Goal

Answer "which of our people are exposed, from where, and how bad" for a monitored domain.

## Steps

1. **Confirm what is monitored.** `GET /watchlist/identifiers` (`eap-list-all-identifiers`). Filter with
   `watchlist_type` (`email`, `domain`, `subdomain`, `ip`) and `verified=yes` to see only verified entries.
   If the domain you were asked about is not on the watchlist, say so — the API will not return records for it.
2. **Pull the exposure set.** `GET /breach/data/watchlist` (`eap-get-all-records-in-watchlist`) for everything
   on the watchlist, or narrow with `GET /breach/data/domains/{domain}` (`eap-get-records-by-domain`).
   Bound the window with `since` / `until` on `spycloud_publish_date`, or `since_modification_date` /
   `until_modification_date` to catch re-published records. Filter with `severity` and `source_id`.
3. **Page through it.** Follow `cursor` until it comes back empty. Stop early if you already have enough to
   answer — 1,000 records per page adds up fast.
4. **Resolve the sources.** Each record carries `source_id`. Call `GET /breach/catalog/{id}`
   (`eap-get-metadata-for-a-breach`) — it accepts comma-delimited IDs — to turn IDs into breach title,
   `breach_main_category` (`combolist` / `breach` / `malware`) and `breach_date`. `GET /breach/catalog`
   (`eap-list-all-breach-metadata`) with `query` searches the catalog by name.
5. **Drill into one person.** `GET /breach/data/emails/{email}` (`eap-get-records-by-email-address`) —
   up to 10 comma-delimited addresses, or their hashes.

## How to read the result

- `breach_main_category: malware` is materially worse than `breach` — a malware record means the device was
  infected, so rotate credentials **and** treat the endpoint as compromised (see the malware-remediation skill).
- `password_type != plaintext` with `password_plaintext` present means SpyCloud cracked the hash. The password
  is effectively public; a hashed-only record is a weaker signal.
- `severity` is the field to sort by when you have to pick what to remediate first.

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
