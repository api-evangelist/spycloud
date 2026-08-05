---
name: Monitor compromised cards by BIN
description: Retrieve compromised credit, debit, gift and loyalty card records recaptured from breaches and malware,
  addressed by six-character BIN.
api: openapi/spycloud-compromised-credit-card-openapi.yml
operations:
- list-credit-cards
- fd-get-credit-cards-by-bin
---

# Monitor compromised cards by BIN

## Goal

Find recaptured cards belonging to BIN ranges you issue, so they can be reissued or watched.

## Steps

1. **Enumerate what is available.** `GET /data/cc/bins` (`list-credit-cards`).
2. **Query your BINs.** `GET /data/cc/bins/{bin}` (`fd-get-credit-cards-by-bin`) — up to **10 comma-delimited
   6-character BINs** per request. Covers credit, debit, gift and loyalty cards.
3. **Bound the window** with `since` / `until` on `spycloud_publish_date`, and page on `cursor`.

## Rules

- Query only BINs your organisation issues. A BIN you do not own is somebody else's cardholder data.
- Newly published records are the actionable ones — run this on a schedule with `since` set to the last run,
  rather than re-pulling history every time.
- Card data is the most sensitive payload on this platform. Move matches straight into the reissue workflow;
  do not stage full card numbers in logs, tickets or model context.

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
