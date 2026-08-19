---
name: Clear or purge the Ezoic CDN cache
description: >-
  Invalidate cached content on the Ezoic CDN after a publish — one URL, a batch of URLs,
  a set of surrogate keys, or the whole domain.
api: openapi/ezoic-cdn-api-openapi.yml
operations: [cdnPing, clearCache, bulkClearCache, clearBySurrogateKeys, purgeCache]
---

# Clear or purge the Ezoic CDN cache

Base URL: `https://api-gateway.ezoic.com` · auth: `?developerKey=...`

Enable the CDN service first under **Settings → API Access** in the Ezoic dashboard; the
key is shared with every other enabled gateway service.

Every endpoint is **POST**, and every endpoint returns the **same envelope** whether it
succeeds or fails:

```json
{"Success":true,"Error":""}
```

Branch on `Success`. `Error` is an empty string on success, not absent or null.

## Choose the narrowest tool that works

1. **One URL changed** — `clearCache` (`POST /cdnservices/clearcache`),
   body `{"url": "https://www.example.com/"}`. Clears that URL in all regions.

2. **A set of URLs changed** — `bulkClearCache` (`POST /cdnservices/bulkclearcache`),
   body `{"urls": ["https://www.example.com/", "https://www.example.com/page1"]}`.
   Processed in **batches of 100** — chunk longer lists yourself.

3. **A content group changed** — `clearBySurrogateKeys`
   (`POST /cdnservices/clearbysurrogatekeys`),
   body `{"keys": "key1,key2,keyC", "domain": "example.com"}`. `keys` is a single
   comma-separated **string**, not an array. This is the right call when one source object
   (an author, a category, a product) appears on many pages.

4. **Everything changed** — `purgeCache` (`POST /cdnservices/purgecache`),
   body `{"domain": "example.com"}`. Purges the entire cache for the domain. Last resort:
   it re-empties a warm cache and pushes the whole load back onto your origin.

5. **Checking liveness** — `cdnPing` (`POST /cdnservices/ping/`). Note the trailing slash,
   which the other four endpoints do not have.

## The `domain` rule

`clearBySurrogateKeys` and `purgeCache` take a **bare** domain. Do **not** include:

- a subdomain (`www.`)
- a scheme (`http://`, `https://`)
- a path (`/`)

So `example.com`, never `https://www.example.com/`. The `url` fields on `clearCache` and
`bulkClearCache` are the opposite — those are full URLs including scheme.

## Rules and gotchas

- A missing `developerKey` returns **HTTP 503 `Developer key is empty.`** — auth failure,
  not an outage.
- **No failure statuses or example error payloads are published**, so you only know the
  envelope shape, not the codes. Log `Error` verbatim.
- **No rate limits are published.** After a bulk publish, prefer one `bulkClearCache` call
  per 100 URLs over 100 `clearCache` calls.
- **This API is unversioned** (`/cdnservices/`, no `/v1`). No breaking-change signal exists.
- These calls are effectively idempotent — clearing an already-clear object is a no-op —
  but no idempotency key is published, so a retry after a timeout is safe by effect rather
  than by contract.
