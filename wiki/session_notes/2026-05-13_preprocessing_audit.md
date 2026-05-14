I inspected `AGENTS.md`, `tasks/data_preprocessing.md`, and all raw Excel files. I have not started coding.

**Dataset Summary**

| File | Likely Role | Shape / Granularity | Key Fields |
|---|---:|---:|---|
| `2.11.26all.xlsx` | NCR All | 10,619 rows, one row per `Notification` | `Notification`, `Supplier`, `Material`, `Created on`, `Notification Date`, `Completion by date`, `Complaint Quantity` |
| `NCR_Closed_Open_Supplier_Vendor_ID.xlsx` | NCR Open & Closed with vendor mapping | 20,599 rows, one row per `Notification` | `Notification`, `Vendor_ID`, `Supplier Name`, `Material Number`, `Notification Created Date`, `Notification Completion Date` |
| `All Open&Closed NCR report.xlsx` | Task-level NCR report / dashboard export | Source sheets repeat notification by task. `Sheet1`: 166,533 rows, 23,731 notifications | `Notification Number`, task fields, disposition, supplier name, dates |
| `Gred 2.9.25-2.9.26.xlsx` | GR History | 60,753 rows, likely GR material-document line level | `Material Document`, `Material Doc.Item`, `Vendor`, `Movement type`, `Posting Date`, `Quantity` |

**Likely Join Keys**

Primary keys:

- NCR event key: `Notification` / `Notification Number`
- Supplier key: normalized `vendor_id`
  - `NCR_Closed_Open_Supplier_Vendor_ID.xlsx`: `Vendor_ID`
  - `2.11.26all.xlsx`: `Supplier`
  - GR History: `Vendor`
- GR line key: `Material Document` + `Material Doc.Item`
- Time key for monthly aggregation: month derived from `Notification Created Date` / `Created on` and GR `Posting Date`

Coverage notes:

- `2.11.26all.xlsx` overlaps vendor-mapped NCR data on 9,277 notifications.
- Of those overlaps, vendor IDs match on 9,275 and mismatch on 2.
- `All Open&Closed NCR report.xlsx` `Sheet1` contains all 20,599 vendor-mapped notifications, plus 3,132 additional notifications.
- GR vendors overlap NCR vendors: 74 shared vendors out of 134 GR vendors and 118 NCR vendors.

**Date Columns**

NCR All:

- `Created on`: 2019-02-19 to 2026-02-11
- `Notification Date`: 2019-02-19 to 2026-02-11
- `Completion by date`: 2019-02-19 to 2026-02-11, 580 missing
- `Changed On`, `Malfunction Start`, `Reference Date`

NCR Open & Closed vendor map:

- `Notification Created Date`: 2019-04-04 to 2026-02-04
- `Notification Completion Date`: 2019-04-11 to 2026-02-04, 828 missing
- `Task Release Date`, `Change Date`, `Task Completion Date`

GR History:

- `Posting Date`: 2025-02-10 to 2026-02-09
- `Document Date`: 2021-12-01 to 2026-02-09

**Supplier / Vendor Mapping Logic**

Recommended logic:

1. Normalize all vendor IDs as strings with no `.0`.
2. Use `Vendor_ID` from `NCR_Closed_Open_Supplier_Vendor_ID.xlsx` where available.
3. For `2.11.26all.xlsx`, treat `Supplier` as `vendor_id`.
4. Merge NCR All and Open & Closed by `Notification`.
5. Prefer event fields from the one-row-per-notification sources over task-level dashboard sheets.
6. Use `Supplier Name` from the vendor-mapped file when available.
7. Preserve supplier-name conflicts for QA rather than resolving by name.

Known mapping issues:

- 3 rows in vendor-mapped NCR data missing `Vendor_ID` and `Supplier Name`.
- 44 rows in NCR All missing `Supplier`.
- A few many-to-many name/id cases exist:
  - `WABTEC Passenger Transit Corpo`
  - `Vapor Stone Rail Systems`
  - `UKM Transit Products`
- Changchun/internal candidates appear in NCR All:
  - `1000200000`
  - `2000004123`
  - `2000044404`

**Likely NCR Event Granularity**

Use one row per `Notification` / `Notification Number`.

Do not use the task-level sheets directly as event rows. `All Open&Closed NCR report.xlsx` repeats notifications heavily because each notification can have multiple task rows. It is useful for task/disposition/status enrichment, but event-level preprocessing must deduplicate or aggregate by notification.

**Likely GR Exposure Granularity**

Raw GR appears to be material-document line level:

- `Material Document` + `Material Doc.Item` is unique for non-null rows.
- Movement types are only `101`, `102`, plus one malformed/blank row.
- `101` quantity total: positive receipts.
- `102` quantity total: negative reversals.
- Vendor exposure should aggregate to `vendor_id + posting_month`.

Recommended output fields:

- `received_qty_101`: sum of `Quantity` where movement type is `101`
- `received_qty_102`: sum of `Quantity` where movement type is `102`, likely negative as stored
- `received_qty_net`: `received_qty_101 + received_qty_102`

**Likely Preprocessing Steps**

1. Load raw Excel files without modifying them.
2. Normalize column names to snake_case.
3. Normalize IDs:
   - notification IDs as strings or integer strings
   - vendor IDs as integer-like strings
   - material IDs as strings
4. Convert Excel serial dates and native Excel dates into proper dates.
5. Build `ncr_event` from the one-row NCR sources.
6. Use task-level Open & Closed report only to validate/enrich, not as the event grain.
7. Validate notification merge coverage and vendor mapping coverage.
8. Build GR exposure from movement types `101` and `102`.
9. Aggregate GR to vendor-month.
10. Aggregate NCR events to vendor-month using created month.
11. Build vendor-month base as the union of vendor-months from exposure and NCR summaries.
12. Write a data quality summary.

**Known Risks / Issues**

- NCR sources have different date cutoffs: GR is only 2025-02-10 to 2026-02-09, while NCR goes back to 2019.
- Task-level NCR report can inflate event counts if not deduplicated.
- Vendor/name mappings are not perfectly one-to-one.
- Two overlapping NCR records have conflicting vendor IDs.
- Material value has many suspicious values:
  - 990 zero values
  - 1,441 penny values
  - very large max around 18.4M
- NCR quantities have extreme values:
  - max 23,861 in vendor-mapped file
  - max 19,960 in NCR All
- GR has 26,490 rows missing `Vendor`; those appear tied to rows without purchase orders and should not enter vendor exposure unless a reliable mapping exists.
- GR has one malformed/blank movement-type row with a very large amount; it should be excluded and flagged.
- Cause responsibility is missing for 1,639 vendor-mapped NCR events.

**Recommended Clean Output Tables**

1. `ncr_event.csv`
   - one row per notification
   - `notification_number`
   - `vendor_id`
   - `supplier_name`
   - `material`
   - `material_description`
   - `created_date`
   - `completion_date`
   - `qty`
   - `material_value`
   - `disposition`
   - `cause_responsibility`
   - `notification_type`
   - `project`
   - source/coverage flags

2. `vendor_month_exposure.csv`
   - `vendor_id`
   - `month`
   - `received_qty_101`
   - `received_qty_102`
   - `received_qty_net`
   - optional QA fields: GR row count, unique materials, unique POs

3. `vendor_month_base.csv`
   - complete vendor-month table
   - exposure fields
   - same-month NCR summaries only
   - no future target
   - suggested NCR summary fields: event count, total qty, total material value, supplier-responsible count, open/closed count

4. `data_quality_summary.md` or `.csv`
   - row counts
   - vendor counts
   - date ranges
   - notification merge coverage
   - missing values
   - duplicate/aggregation warnings
   - known mapping conflicts