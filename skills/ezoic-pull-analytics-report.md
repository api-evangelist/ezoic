---
name: Pull an Ezoic analytics report
description: >-
  Pull revenue, traffic and engagement data out of an Ezoic account with the Big Data
  Analytics API — list the predefined reports, read one report's definition, and page
  through its rows.
api: openapi/ezoic-big-data-analytics-api-openapi.yml
operations: [getReports, getReport, getData, getColumns]
---

# Pull an Ezoic analytics report

Base URL: `https://api-gateway.ezoic.com`

## Before you start

Big Data Analytics is **off until it is switched on**. In the Ezoic dashboard open
**Settings → API Access**, turn on the service, and copy the API key. That key is your
`developerKey` and it is shared with every other Ezoic gateway service you have enabled.

## Authentication (every request)

- `developerKey` as a **query parameter**. There is no header form and no OAuth for this API.
- The gateway validates the key, confirms the service is enabled for it, and confirms any
  `DomainId` you use belongs to the account.
- A missing key returns **HTTP 503 with the body `Developer key is empty.`** — that is an
  auth failure, not an outage. Do not retry it as a transient error.

## Steps

1. **List the predefined reports** — `getReports` (`GET /bdaservices/getreports/`).
   Returns report names, for example `revenueDaily`.

2. **Read the report definition** — `getReport`
   (`GET /bdaservices/getreport/?reportName=revenueDaily`).
   Check `DateBaseSelectId`. If it is `BASE_NOT_SELECTED` — which is the case for most
   predefined reports — the report has **no built-in date range** and step 3 must supply
   `StartDate` and `EndDate`.

3. **Pull the rows** — `getData` (`POST /bdaservices/getdata/?reportName=revenueDaily`)
   with a JSON body:

   ```json
   {
     "StartItem": 0,
     "MaxItems": 10,
     "Platform": "ALL",
     "DomainId": 9999,
     "DateGrouping": "DAILY",
     "SegmentIds": [1],
     "StartDate": "2018-10-01",
     "EndDate": "2018-10-01"
   }
   ```

   - `MaxItems` is **required**.
   - `Platform` is `EZOIC`, `ORIG`, or `ALL` (combined).
   - `DateGrouping` controls day / week / month grouping.
   - `SegmentIds` is optional; `[1]` is the premade "All Users" segment.

4. **Paginate** by incrementing `StartItem` by `MaxItems`. Some reports — a URL dimension
   especially — return thousands of rows. There is **no total count and no cursor** in the
   response, so keep paging until a page comes back short.

5. **Discover columns** when you need to know what is available — `getColumns`
   (`GET /bdaservices/getcolumns/`). Column availability is account-dependent; not every
   column exists on every account.

## Rules and gotchas

- **No published error contract.** Ezoic documents only successful payloads for this API.
  Treat any non-2xx, or any response missing the expected data, as a failure and log the
  raw body.
- **No rate limits are published**, and there are no `RateLimit-*` headers. Pace your own
  calls.
- **This API is unversioned.** There is no `/v1` in the path, so there is no published
  mechanism warning you about a breaking change. Pin nothing; validate the shape you get.
- **Casing is inconsistent** — request fields are UpperCamel (`StartItem`, `DomainId`),
  column values are snake_case (`report_day`, `engaged_page_pageviews`), and the paths mix
  styles (`getdata/` next to `getCustomData/`).
- Reporting for an agent instead? The **Ezoic Analytics MCP server**
  (`https://analytics-mcp.ezoic.com/mcp`, OAuth) reads the same warehouse read-only. See
  `mcp/ezoic-tool-crosswalk.yml`.
