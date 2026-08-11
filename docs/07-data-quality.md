# 7 — Data Quality: Partitioning as an EDA Tool

An unexpected bonus of partitioning: it surfaces data anomalies that are
invisible in a flat table.

## 7.1 The discovery

The NYC Taxi dataset (June–October 2023) was loaded into `trips_by_month`,
partitioned by `months(tpep_pickup_datetime)`. Aggregating by partition
revealed:

```
| partition | record_count | decodes to | note               |
|  {395}    |            5 | 2002-12    | timestamp error    |
|  {396}    |            1 | 2003-01    | timestamp error    |
|  {467}    |           10 | 2008-12    | timestamp error    |
|  {468}    |           11 | 2009-01    | timestamp error    |
|  {640}    |           19 | 2023-05    | outside load range |
|  {641}    |    3,307,234 | 2023-06    | real data          |
|  {642}    |    2,907,084 | 2023-07    | real data          |
|  {643}    |    2,824,199 | 2023-08    | real data          |
|  {644}    |    2,846,740 | 2023-09    | real data          |
|  {645}    |    3,522,242 | 2023-10    | real data          |
|  {646}    |           13 | 2023-11    | outside load range |
```

**59 rows** out of 15,407,558 (0.0004%) have timestamps from 2002, 2003,
2008, 2009, or months outside the loaded range.

## 7.2 Why partitioning makes this visible

In an **unpartitioned** table, these 59 bad rows are sprinkled across the
17 data files and disappear into the noise. In a **partitioned** table,
each bad row's transform output creates its **own tiny partition**:

```
Unpartitioned:        Monthly partitioned:
┌─────────────┐       ┌──────────────────────┐
│  17 files,  │       │ 2023-06: 4 files     │
│  2002+2008  │  ──►  │ 2023-07: 3 files     │
│  rows mixed │       │ ...                  │
│  in         │       │ 2002-12: 1 file (5)  │ ← sticks out
└─────────────┘       │ 2008-12: 1 file (10) │ ← sticks out
                      └──────────────────────┘
```

Grouping by partition value is effectively a histogram over the transform
column — anomalies form their own bars.

## 7.3 Root cause (NYC Taxi specifics)

NYC Taxi trip data is collected automatically from electronic taxi meters.
The stray timestamps come from:

- **Meter resets** to a manufacturing default date (2002, 2008...).
- **GPS/clock faults** on the device.
- **Manual entry errors** by drivers.
- A few **refund/correction records** with bad metadata.

TLC's own pipeline filters most of these; a handful always slip through.

### Inspecting the bad rows

```sql
SELECT tpep_pickup_datetime, tpep_dropoff_datetime,
       trip_distance, total_amount, PULocationID
FROM polaris.taxi.trips_by_month
WHERE YEAR(tpep_pickup_datetime) < 2020;
```

Typical red flags in the result: `trip_distance = 0`, `total_amount = 0`,
`PULocationID = 0` (NULL encoding), `pickup > dropoff`.

## 7.4 Two kinds of "skew" — don't confuse them

When partition counts differ a lot, ask which kind of skew it is:

### Type 1: Data-quality skew (a bug)

A few partitions with single-digit row counts next to partitions with
millions → **bad data**. Action: audit and clean, or at least document.

### Type 2: Seasonal/business skew (normal)

Even the legitimate months differ noticeably:

```
August 2023:  2,824,199 rows  →  ~91,100 trips/day  (lowest)
October 2023: 3,522,242 rows  → ~117,400 trips/day  (highest)
```

That ~29% gap is **NYC seasonality**, not a data error:

- July–August: summer heat, residents leave the city → fewer trips.
- September: schools reopen, people return.
- October: peak tourism season, great weather, events → highest volume.

Different number of days per month also contributes (30 vs 31), but
seasonality dominates.

## 7.5 A reusable audit pattern

```sql
-- Histogram of rows per partition; sort by row count to surface anomalies
SELECT
    data_file.partition,
    SUM(data_file.record_count) AS records,
    COUNT(*) AS files
FROM polaris.<namespace>.<table>.entries
GROUP BY data_file.partition
ORDER BY records DESC;
```

Any partition with a wildly smaller row count than its neighbors is worth
investigating. This works for time, bucket, and truncate partitions alike.

## 7.6 Other anomalies visible in the NYC Taxi data

Beyond timestamps, the `readable_metrics` bounds revealed extreme
`total_amount` outliers:

```
file 6:  total_amount  -556.55  ..  386,987.63   ← impossible fare
file 15: total_amount  -872.75  ..  187,513.90   ← impossible fare
```

Negative values are refunds/errors; the huge positives are data-entry
mistakes. These bounds also explain **why metric pushdown works so well**:
a query `WHERE total_amount > 50000` skips almost every file because their
`upper_bound` is well under 50,000.

## Key takeaways

- Partitioning is an **accidental EDA tool**: bad values form their own
  tiny partitions and stand out immediately.
- Always distinguish **data-quality skew** (a bug) from **seasonal skew**
  (a business pattern).
- Use the partition-histogram query as a quick data audit.
- `readable_metrics` min/max bounds reveal both outliers and the reason
  metric pushdown is effective.

Next: [08-cheatsheet.md](08-cheatsheet.md)
