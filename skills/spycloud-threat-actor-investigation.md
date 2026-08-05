---
name: Pivot across recaptured data to profile a threat actor
description: Run a multi-selector investigation across the SpyCloud Cybercrime Investigations API, pivoting between
  emails, usernames, passwords, IPs, devices, social handles and identity documents.
api: openapi/spycloud-investigations-openapi.yml
operations:
- get-records-by-email-address
- inv-get-records-by-username
- inv-get-records-by-password
- inv-get-records-by-ip-address
- inv-get-records-by-infected-machine-id
- inv-get-records-by-log-id
- inv-get-records-by-social-handle
- inv-get-records-by-domain
- inv-get-records-by-phone-number
- inv-get-records-by-email-username
- inv-list-all-breach-metadata
- inv-get-metadata-for-a-breach
---

# Pivot across recaptured data to profile a threat actor

## Goal

Start from one selector and build out an actor profile by pivoting on what each result reveals.

## Steps

1. **Seed the investigation** with whatever you have:
   `GET /breach/data/emails/{email}` (`get-records-by-email-address`),
   `GET /breach/data/usernames/{username}` (`inv-get-records-by-username`),
   `GET /breach/data/domains/{domain}` (`inv-get-records-by-domain`),
   `GET /breach/data/ips/{ip}` (`inv-get-records-by-ip-address`, CIDR with `_` for `/`),
   `GET /breach/data/phone-numbers/{phone_number}` (`inv-get-records-by-phone-number`),
   or `GET /breach/data/social-handles/{social_handle}` (`inv-get-records-by-social-handle`).
2. **Pivot on the local part.** `GET /breach/data/email-usernames/{email_username}`
   (`inv-get-records-by-email-username`) matches the portion before the `@` across providers — this is the
   single highest-yield pivot for finding an actor's other addresses.
3. **Pivot on the password.** `GET /breach/data/passwords/{password}` (`inv-get-records-by-password`) with
   `fuzzy=true` to catch near-variants (searching `Examplepassword12` also finds close forms). Password reuse
   is what links personas.
4. **Pivot on the machine.** `GET /breach/data/infected-machine-ids/{infected_machine_id}`
   (`inv-get-records-by-infected-machine-id`) and `GET /breach/data/log-ids/{log_id}`
   (`inv-get-records-by-log-id`) collapse everything harvested from one infected host or one malware log.
5. **Resolve sources.** `inv-list-all-breach-metadata` / `inv-get-metadata-for-a-breach`.

## Rules

- **One pivot at a time, and record why.** Each hop widens the identity set; without recording the linking
  field you will produce a profile you cannot defend.
- Identity-document selectors (`inv-get-records-by-passport-number`, `-drivers-license`, `-national-id`,
  `-social-security-number`, `-cc-number`, `-bank-number`) each have their own hashing rules — read the
  parameter description before hashing. Several require a SHA-1 of normalized plaintext first, then a
  SHA-256/512 of that.
- A 422 means the selector matched more than 2,000 records. That is a signal the selector is too generic,
  not an error to retry.
- For automated identity correlation instead of manual pivoting, use the IDLink skill.

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
