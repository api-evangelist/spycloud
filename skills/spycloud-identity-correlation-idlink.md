---
name: Correlate an identity across recaptured data with IDLink
description: Use the SpyCloud IDLink API to expand one email, phone number or username into a correlated identity
  graph up to four pivot levels deep.
api: openapi/spycloud-idlink-openapi.yml
operations:
- idl-get-records-by-email
- idl-get-records-by-phone-number
- idl-get-records-by-username
---

# Correlate an identity across recaptured data with IDLink

## Goal

Turn a single selector into the identity SpyCloud believes sits behind it — automatically, instead of pivoting by hand.

## Steps

1. **Pick the entry point.** `GET /query/emails/{email}` (`idl-get-records-by-email`),
   `GET /query/phone-numbers/{phone}` (`idl-get-records-by-phone-number`, 7-15 digits or a hash), or
   `GET /query/usernames/{username}` (`idl-get-records-by-username`).
2. **Set `max_depth` deliberately — it is required.** `1` is a plain query. `2` adds one pivot, `3` two, `4` three.
   Start at 1 or 2. Depth is where precision goes to die: every additional hop pulls in identities linked by
   weaker evidence, so a depth-4 result is a lead list, not a conclusion.
3. **Set `output_format` — also required.** `json` for records, `json-graph-spec` when you want the graph
   structure to render or traverse.

## Rules

- All three parameters (`max_depth`, `output_format`, and the selector) are required. A 400 usually means one
  is missing, not that the identity is unknown.
- Use IDLink for synthetic-identity detection and holistic exposure scoring; use Investigations when you need
  to control and justify each individual pivot.
- Report the depth alongside the answer. "Correlated at depth 1" and "correlated at depth 4" are different
  claims and must not be presented the same way.

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
