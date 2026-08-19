---
name: Build a custom Ezoic report with a segment
description: >-
  Compose a segment from Ezoic's multifilters, build a saved custom report over chosen
  dimension and metric columns, and pull its data — or skip the save and query columns
  inline with getCustomData.
api: openapi/ezoic-big-data-analytics-api-openapi.yml
operations: [getColumns, getMultiFilters, getMultiFilterTypes, createSegment, createCustomReport, getCustomReports, getData, getCustomData, getSegments]
---

# Build a custom Ezoic report with a segment

Base URL: `https://api-gateway.ezoic.com` · auth: `?developerKey=...` (see
`authentication/ezoic-authentication.yml`).

There are two routes. Take the second one unless you need the report to persist.

## Route A — a saved custom report

1. **Find your columns** — `getColumns` (`GET /bdaservices/getcolumns/`). Dimensions group
   rows (`report_day`), metrics measure them (`visits`, `pageviews`, `revenue`, `epmv`,
   `engaged_time`, `bounce_rate`, `copy_paste_per_pageview`).

2. **Build a segment** (optional). Segments split data into categories.
   - `getSegments` (`GET /bdaservices/getsegments/`) lists what already exists, including
     premade ones — "All Users" (id `1`, no filter) and "Desktop Traffic".
   - `getMultiFilters` (`GET /bdaservices/getmultifilters/`) lists the preset filter groups
     (country, device, gender, site). Country comes back as
     `{"FilterName":"Country","MultiFilterId":1,"MultiFilterTypeId":1}`.
   - `getMultiFilterTypes` (`GET /bdaservices/getmultifiltertypes/`) gives the option keys
     for each type. Type `1` is Country, and its keys are two-character country codes.
   - `createSegment` (`POST /bdaservices/createsegment/`):

     ```json
     { "SegmentName": "US and Canada",
       "SegmentMultiFilters": { "1": { "FilterValues": ["US", "CA"] } } }
     ```

     The response carries the segment's `id`. **This POST is not idempotent** — calling it
     twice creates two segments.

3. **Create the report** — `createCustomReport`
   (`POST /bdaservices/createcustomreport/`). Set `ReportDateRange.BaseSelectId` (for
   example `BASE_LAST_7`) so later `getData` calls need no dates. `Charts[]` carries
   `DimensionColumns`, `MetricColumns`, `Segments` and `Order`
   (`{"ColumnNumber": 0, "Direction": "DESC"}` — sorted by column **index**, not name).
   Returns the custom report id. Also not idempotent.

4. **Pull it** — `getData`
   (`POST /bdaservices/getdata/?customReportId=<id>`) with `StartItem`, `MaxItems`,
   `Platform`, `DomainId`, `DateGrouping`, and `SegmentIds` when you want to override.
   Lost the id? `getCustomReports` (`GET /bdaservices/getcustomreports/`) lists them all.

   **Segment precedence:** a custom report's embedded segments apply when the `getData`
   body supplies none; if the body supplies segments, the body wins.

## Route B — no saved report

`getCustomData` (`POST /bdaservices/getCustomData/`) does it in one call. `StartDate`,
`EndDate`, `DimensionColumns` and `MetricColumns` are required; `Filters[]` applies row
filters (`{"Type":"INCLUDE","FilterKey":"epmv","OperationId":"GREATER","FilterValue":"3"}`),
`RevenueDecimalPlaces` takes 2–6 for USD metrics, and omitting both `StartItem` and
`MaxItems` (or setting both to `0`) returns **all** rows.

Note the endpoint casing: `getCustomData` is camelCase where the rest of the service is
lowercase.

## Rules and gotchas

- `SegmentIds` defaults to All Users (`[1]`) when empty.
- `DomainId` may be omitted on `getCustomData` to span every site on the account.
- No error contract, no rate limits, no versioning — see
  `skills/ezoic-pull-analytics-report.md` and `errors/ezoic-problem-types.yml`.
- **Two of these operations write.** `createSegment` and `createCustomReport` mutate the
  account and publish no idempotency key, so a retried request after a timeout leaves a
  duplicate behind. Read back with `getSegments` / `getCustomReports` before retrying.
