# 6G-DALI Input Dataset Format

**Version:** 0.2 (draft)
**Project:** 6G-DALI (SNS-JU)
**Companion documents:** [External Dataset Contribution Guide](External-Dataset-Contribution-Guide.md) · [6G-DALI Metadata Application Profile](6G-DALI-Metadata-Application-Profile.md)

---

## Table of Contents

1. [Purpose and scope](#1-purpose-and-scope)
   - 1.1 [Two kinds of dataset — and which rules apply to yours](#11-two-kinds-of-dataset--and-which-rules-apply-to-yours)
   - 1.2 [You do not have to get everything right on the first attempt](#12-you-do-not-have-to-get-everything-right-on-the-first-attempt)
2. [Accepted file formats](#2-accepted-file-formats)
   - 2.1 [Archives must be expanded before upload](#21-archives-must-be-expanded-before-upload)
   - 2.2 [Formats not yet supported](#22-formats-not-yet-supported)
3. [File-level requirements](#3-file-level-requirements)
   - 3.1 [Encoding: UTF-8, no byte-order mark](#31-encoding-utf-8-no-byte-order-mark)
   - 3.2 [Delimiters and decimal separator](#32-delimiters-and-decimal-separator)
   - 3.3 [Compression](#33-compression)
   - 3.4 [Size and one-file-per-distribution](#34-size-and-one-file-per-distribution)
4. [Column naming](#4-column-naming)
   - 4.1 [Names must match your declared metadata exactly](#41-names-must-match-your-declared-metadata-exactly)
   - 4.2 [Use ASCII lowercase `snake_case`](#42-use-ascii-lowercase-snake_case)
5. [Table structure requirements](#5-table-structure-requirements)
   - 5.1 [Missing values are reported, not rejected](#51-missing-values-are-reported-not-rejected)
   - 5.2 [One column = one measured variable](#52-one-column--one-measured-variable)
   - 5.3 [One row = one point in time (time series only)](#53-one-row--one-point-in-time-time-series-only)
6. [Timestamps](#6-timestamps)
   - 6.1 [Declare the timestamp column explicitly](#61-declare-the-timestamp-column-explicitly)
   - 6.2 [If you don't, it is guessed from the header](#62-if-you-dont-it-is-guessed-from-the-header)
   - 6.3 [The values must also look like timestamps](#63-the-values-must-also-look-like-timestamps)
   - 6.4 [Regular sampling interval](#64-regular-sampling-interval)
7. [Worked examples](#7-worked-examples)
   - 7.1 [Fixing an embedded-unit column](#71-fixing-an-embedded-unit-column)
   - 7.2 [Pivoting a multi-source (long-format) dataset to wide format](#72-pivoting-a-multi-source-long-format-dataset-to-wide-format)
   - 7.3 [Anti-pattern: many time series stacked in one file](#73-anti-pattern-many-time-series-stacked-in-one-file)
   - 7.4 [What the platform will accept and handle in future](#74-what-the-platform-will-accept-and-handle-in-future)
8. [Pre-submission checklist](#8-pre-submission-checklist)
9. [Support](#9-support)

---

## 1. Purpose and scope

The [External Dataset Contribution Guide](External-Dataset-Contribution-Guide.md) describes *what metadata to submit and how*. This document answers a narrower, purely technical question: **what must the data file itself look like** for the 6G-DALI validation pipeline to accept it — and, more importantly, for the downstream DataOps services (quality assessment, gap detection, imputation, aggregation) to produce correct results rather than merely run without error.

A file can be perfectly well-formed CSV and still be hard to use downstream, because *"parses fine"* and *"is a well-formed dataset"* are different bars.

This document is written for contributors converting an existing dataset — collected under whatever conventions their own testbed or tooling used — into a form 6G-DALI can validate and process. Requirements are stated normatively: **must** is enforced by the pipeline and blocks promotion to the Community Datasets Catalogue; **should** is a strong recommendation that avoids known failure modes but is not itself blocking.

### 1.1 Two kinds of dataset — and which rules apply to yours

6G-DALI accepts **both time-series and non-temporal tabular datasets**. The distinction matters because it decides which half of this document you need:

| | **Tabular** (non-temporal) | **Time series** |
|---|---|---|
| What it is | Each row is an observation of an entity, with no inherent time ordering | Each row is an observation of one or more variables at a point in time |
| Examples | Geographical coverage of network equipment; site and antenna inventories; device or subscriber records; configuration sweeps; benchmark or simulation result sets; labelled pattern datasets | RAN measurement traces; link and probe monitoring; KPI histories; resource-utilisation logs |
| Quality checks applied | Completeness, type and range checks, outlier detection, duplicate and primary-key checks | All of the tabular checks, **plus** gap detection, ordering and interval checks, and imputation |
| Which sections apply | [§2](#2-accepted-file-formats)–[§4](#4-column-naming), [§5.1](#51-missing-values-are-reported-not-rejected)–[§5.2](#52-one-column--one-measured-variable) | The same, **plus** [§5.3](#53-one-row--one-point-in-time-time-series-only) and [§6](#6-timestamps) |

**If your dataset has no meaningful time axis, that is fine** — it is a first-class case, not a degraded one. Submit it, say so on the submission form, and simply skip [§5.3](#53-one-row--one-point-in-time-time-series-only) and [§6](#6-timestamps); everything else still applies. A non-temporal dataset is validated against the tabular check suite and is equally eligible for the catalogue and for DataOps services.

Datasets whose value lies in something other than rows and columns — packet captures, raw I/Q samples, images — are a separate conversation; see [§2.2](#22-formats-not-yet-supported).

### 1.2 You do not have to get everything right on the first attempt

The rules below are worth following because they make your dataset genuinely more useful to everyone who picks it up, not because 6G-DALI wants to turn contribution into paperwork. Two things are worth knowing before you start:

- **Most issues are reported, not fatal.** Only a few things actually block publication — an unparseable file, a declared column that isn't there, an empty column or file. Missing values, irregular sampling and imperfect naming are recorded in the quality report and travel with the dataset as useful information about it.
- **These requirements will loosen over time.** They describe the current release, not a permanent contract. Future versions of the platform will both **accept a wider range of datasets** — more formats, archives, larger and multi-file datasets, long-format time series, and non-tabular data — and **do more of the preparation for you**. See [§7.4](#74-what-the-platform-will-accept-and-handle-in-future).

If in doubt, **talk to us before investing effort in a conversion** ([Section 9](#9-support)). A five-minute exchange is usually cheaper than a rewrite. And if your dataset does not fit what we accept today, we would still rather hear about it than not — what arrives in this round decides what the next release accepts.

---

## 2. Accepted file formats

The pipeline determines a file's format from its **filename extension** and parses it accordingly:

| Extension | Format | Notes |
|---|---|---|
| `.csv` | Comma-separated values | **Preferred.** Also the fallback for any unrecognised extension. |
| `.tsv` | Tab-separated values | Tab delimiter. |
| `.json` | JSON | A single array of records (objects), all sharing the same keys. |
| `.jsonl` / `.ndjson` | JSON Lines | One JSON object per line, all sharing the same keys. |

**The extension must be correct, not merely present.** It is the only thing that determines how the file is read; the contents are never inspected to second-guess it. An extension outside the table above is treated as CSV, so a tab-separated file submitted as `data.txt` — or a CSV mistakenly named `data.json` — fails with a confusing parse error rather than a clear "unsupported format" message. **Name every file with the extension that matches its actual contents.**

### 2.1 Archives must be expanded before upload

**Do not submit `.zip`, `.tar`, `.tar.gz`, `.7z` or any other archive.** Archives are not opened by the validation pipeline: an archive is treated as a single opaque file, fails to parse, and blocks the whole submission — including the data files inside it, which are never seen.

**Expand the archive locally and upload the data files themselves.** Each file is then registered and validated as its own distribution of the dataset, with its own quality report ([§3.4](#34-size-and-one-file-per-distribution)).

This applies equally to single-file compression (`.csv.gz`, `.csv.bz2`, `.csv.zst`) — decompress to plain `.csv` before uploading. See [§3.3](#33-compression).

If expanding the archive produces a dataset too large to upload, contact the data team — see [Section 9](#9-support). Splitting a dataset across several uncompressed files is always preferable to submitting one archive.

### 2.2 Formats not yet supported

Only the four formats in the table above are currently read by the validation pipeline. Anything else — **Parquet, ORC, Excel (`.xls` / `.xlsx`), HDF5, Feather/Arrow, PCAP, and binary or instrument-specific formats** — must be **converted to CSV before submission**.

> **Roadmap.** Support for additional formats, beginning with **Apache Parquet**, is planned for a future release of the 6G-DALI platform; columnar formats are a natural fit for the larger measurement datasets in the data space. Until that support ships, a Parquet file cannot be quality-validated or used as a DataOps pipeline input, so please convert. This document will be updated as formats are added — check that you have the current version before preparing a conversion, and see [§7.4](#74-what-the-platform-will-accept-and-handle-in-future) for the fuller picture of what the platform will accept next.

Converting from Parquet or Excel is usually a one-liner; the rules in the rest of this document apply to the *result* of that conversion, so it is worth reading [Section 3](#3-file-level-requirements) onwards before exporting:

```python
import pandas as pd

pd.read_parquet("measurements.parquet").to_csv(
    "measurements.csv", index=False, encoding="utf-8"
)
```

**Non-tabular data** (packet captures, raw I/Q samples, images, logs) does not fit the tabular validation model at all. It can still be contributed to the data space, but the route differs — **contact the data team before submitting** ([Section 9](#9-support)) rather than attempting a conversion that may lose the properties that make the data useful.

---

## 3. File-level requirements

### 3.1 Encoding: UTF-8, no byte-order mark

Files **must** be encoded in **UTF-8**. The pipeline decodes strictly and does not attempt to detect or fall back to other encodings — a file saved as Latin-1, CP1252 or UTF-16 fails to open at all if it contains any non-ASCII character (accented names, `µs`, `°`, `Ω`, an en dash in a header).

The file **must not** begin with a UTF-8 byte-order mark (BOM). A BOM is silently absorbed into the *first column's name*, so a header that visually reads `timestamp` is actually `﻿timestamp` and will not match the `timestamp` you declared in your metadata. This is a common artefact of exporting from Microsoft Excel — in Excel, choose *CSV UTF-8* and verify, or re-save the file with a tool that writes no BOM.

Pure-ASCII content sidesteps both problems entirely and is the safest choice for column names.

### 3.2 Delimiters and decimal separator

The pipeline does **not** sniff the delimiter. It applies the delimiter implied by the extension:

| | Field delimiter | Decimal separator |
|---|---|---|
| `.csv` | comma `,` | period `.` |
| `.tsv` | tab | period `.` |

A **semicolon-delimited** file named `.csv` — the default CSV export in many European locales — parses as a single column containing the whole row as text, and fails every subsequent check for reasons that will not obviously point back to the delimiter. Likewise, **comma decimal separators** (`12,5` meaning twelve and a half) are not interpreted; either they shift your columns or they turn a numeric column into text.

**Check your locale settings before exporting.** Values must use `1234.56`, never `1234,56` and never thousands separators (`1 234,56`, `1,234.56`).

Text fields containing a comma must be quoted with double quotes (`"..."`), following standard CSV quoting rules.

### 3.3 Compression

Submit every data file **uncompressed and unarchived** — see [§2.1](#21-archives-must-be-expanded-before-upload). Neither archives (`.zip`, `.tar.gz`) nor single-file compression (`.csv.gz`) are opened by the validation pipeline; both fail as unparseable files.

Datasets too large to upload uncompressed should be discussed with the data team before submission — see [Section 9](#9-support).

### 3.4 Size and one-file-per-distribution

The portal accepts uploads of up to **1 GB** per session; contact the data team about anything larger ([Section 9](#9-support)). Each data file is validated **independently**, as its own distribution with its own declared column list. A dataset split across several files (e.g. one file per day or per node) is fine, but every file must satisfy every rule in this document on its own, and files that share a schema must share it *exactly* — same column names, same order, same types.

---

## 4. Column naming

### 4.1 Names must match your declared metadata exactly

Every tabular file **must** have a header row of column names (for JSON/JSONL: every record must carry the same keys). Those names **must** match — **exactly and case-sensitively** — the `Dataset columns` list submitted in the metadata form.

This is enforced, not stylistic. The quality checks are generated from your declared column list: each declared column produces both a *column exists* check and a *completeness* (null count) check. A mismatch between a declared name and the file's actual header therefore fails the presence check before a single value is inspected — a declared column the pipeline cannot find is indistinguishable from a column that is genuinely absent. (The completeness result is reported rather than gating; see [§5.1](#51-missing-values-are-reported-not-rejected).)

### 4.2 Use ASCII lowercase `snake_case`

Before processing, the pipeline normalises every column name: it lowercases it, replaces every run of non-alphanumeric characters with a single underscore, and trims leading/trailing underscores.

| Original header | Normalised to |
|---|---|
| `RSRP (dBm)` | `rsrp_dbm` |
| `node-1` | `node_1` |
| `node-1__rsrp` | `node_1_rsrp` |
| `Latency [ms]` | `latency_ms` |

Two practical consequences:

- **Distinctions that survive only in punctuation or case are lost.** `node-1__rsrp` and `node_1_rsrp` both normalise to `node_1_rsrp` — a silent collision between what you intended as two different variables. So do `RSRP` and `rsrp`.
- **Names that differ only by punctuation are not distinct names.** Do not rely on single vs. double underscores, or on hyphens, to separate name parts.

**Name columns in ASCII lowercase `snake_case` from the start** (`node_1_rsrp`, `cpu_usage_pct`, `latency_p95_ms`). Then the name you declare in metadata, the name in the file, and the name used throughout processing are all the same string, and nothing can collide or drift.

---

## 5. Table structure requirements

### 5.1 Missing values are reported, not rejected

**Sparse columns are acceptable.** Real data has gaps — dropped samples, UE disconnections, probes that came online late, survey fields nobody filled in, attributes that simply do not apply to every row — and 6G-DALI does not ask you to hide them. Submit the data as it was collected.

Every column you declare in your metadata is checked for null values, and the result is recorded in the dataset's **quality report** as a completeness measurement, visible to you and to downstream consumers. Empty cells, `NA`, `NULL`, `NaN` and blank strings all count as null. This is an annotation on the dataset, not a gate on it.

Two things follow:

- **Report the gaps honestly in your metadata.** A completeness figure that matches what the `Description` and provenance fields already led a consumer to expect is far more useful than an unexplained one. If a column is sparse by design rather than by accident, say so.
- **Missing values are exactly what the DataOps services are for.** Once published, a dataset can be run through imputation to produce a derived, gap-filled version registered alongside the original — the source data is never modified. If the gaps are the thing you want fixed, publishing first and remediating afterwards is the intended route, not a workaround.

Do not pad gaps with sentinel values (`-1`, `0`, `999`, `"N/A"`) to make a column look complete. A sentinel is indistinguishable from a real measurement to every downstream consumer, silently corrupts any statistic computed over the column, and defeats the imputation that would otherwise have filled the gap properly. **Leave the cell empty** — see [§5.2](#52-one-column--one-measured-variable).

> **Note.** A column that is *entirely* null, or a file with no data rows at all, is treated as a critical data-quality issue rather than a completeness observation, and does block promotion. Drop empty columns before submitting.

### 5.2 One column = one measured variable

A value column must hold the value, and nothing else.

- **No embedded units.** `"2048M"`, `"12ms"`, `"85%"` all break type and range checks the same way: the column no longer parses as numeric at all, so no numeric check can run on it, and any downstream step that expects a number (aggregation, imputation, bounds checking) fails on contact. Move the unit into the column name (`ram_limit_mb`) or, better, into the unit annotation on that variable's metadata entry.
- **No composite values.** A cell holding `"lat=48.85,lon=2.35"` or a JSON blob is one column doing the work of several. Split it.
- **No mixed types.** A numeric column that carries sentinel strings (`"N/A"`, `"-"`, `"unknown"`) for missing values is a text column, so no numeric check can run on it. Use a genuinely empty cell instead — see [§5.1](#51-missing-values-are-reported-not-rejected).

### 5.3 One row = one point in time (time series only)

> **This section applies only to time-series datasets.** If your data has no time axis, skip to [§7](#7-worked-examples) — though the long-format problem described here has a direct tabular analogue, noted at the end.

For time-series datasets, the pipeline requires **exactly one row per timestamp**, in **ascending time order**:

- **Timestamps must be unique.** A repeated timestamp is treated as a data-quality defect, not as "several readings at that instant". Gap detection and imputation both require a single unambiguous value per point in time.
- **Rows must be sorted ascending by timestamp.** Out-of-order rows are flagged. Sort before export.

The uniqueness rule is where **long-format** exports fail, and this is a frequent reason a submission cannot be processed as-is. Many testbeds and monitoring stacks naturally emit one row per `(timestamp, source)` pair, with an identifier column saying which source each row belongs to.

**How to spot it in your own file:** look for a column whose values *repeat in a cycle* rather than varying like a measurement — `node_id`, `sensor_id`, `link`, `probe`, `ue_id`, `cell_id`, `interface`, `flow_id`, `experiment_id`. If your file has one, it almost certainly holds several time series stacked on top of each other, and the row count will be a near-exact multiple of the number of distinct timestamps. A quick check:

```python
df.groupby("timestamp").size().max()     # > 1 means the file is long-format
```

Such a file must be **pivoted to wide format**, or **split into one file per series**, before submission. See the worked example in [§7.2](#72-pivoting-a-multi-source-long-format-dataset-to-wide-format) and the full-scale case in [§7.3](#73-anti-pattern-many-time-series-stacked-in-one-file).

Note that an identifier column is only a problem when it *multiplies* the timeline. A column holding constant metadata (the same `testbed_id` on every row) is harmless, if redundant — that value belongs in the metadata record rather than in a column.

**The tabular equivalent.** Non-temporal datasets have no timestamp to be unique, but they do have an identifying key — a site ID, a cell ID, a device serial, or a combination of columns. The pipeline detects a likely primary key automatically and reports duplicates against it. The same principle applies: each row should describe **one entity**, with the columns describing that entity, and a genuine duplicate is a defect rather than a second reading.

---

## 6. Timestamps

> **This section applies only to time-series datasets.** A non-temporal tabular dataset needs no timestamp column, and nothing here applies to it — see [§1.1](#11-two-kinds-of-dataset--and-which-rules-apply-to-yours). A column that merely records *when a row was written* does not make a dataset a time series.

Time-series validation — gap detection, ordering checks, imputation — only activates if the pipeline can resolve a timestamp column. Getting this right is the single highest-value thing a contributor can do, because when it fails, the run does not error: it **silently falls back to tabular-only validation**, and the dataset is published without any of the time-series quality checks having run.

### 6.1 Declare the timestamp column explicitly

The most reliable option is to **state on submission that the dataset is a time series, and name the column that carries the time axis**, rather than relying on detection. Both are fields on the submission form. Do this whenever the dataset is a time series, and without exception when it is multi-source or irregularly sampled.

Declaring these removes all the guesswork described in [§6.2](#62-if-you-dont-it-is-guessed-from-the-header) and [§6.3](#63-the-values-must-also-look-like-timestamps), and — more importantly — converts a *silent* degradation into a visible error: if you say the dataset is a time series and the named column cannot serve as one, you get told, instead of quietly receiving a report with no time-series checks in it.

### 6.2 If you don't, it is guessed from the header

If no timestamp column is declared, one is inferred from the header. Names are matched case- and punctuation-insensitively, in this priority order:

```
timestamp, time_stamp, event_timestamp, event_time, datetime, date_time, time, ts,
epoch, epoch_time, unix_time, measured_at, observed_at, recorded_at, collected_at,
logged_at, date, day, created_at, updated_at, inserted_at
```

…followed by any column whose name merely *ends in* `_timestamp`, `_datetime`, `_time`, `_date`, or `_at`.

Note the deliberate ordering: an explicit `timestamp` beats a bare `time`, which beats a date-only column, which beats bookkeeping columns like `created_at` — those record when the *row was written*, not when the *measurement was taken*. If your file contains both a measurement time and a record-insertion time, name them so the distinction is unambiguous.

**Recommendation: name the column `timestamp`.** It is the top-ranked match and reads correctly to a human.

### 6.3 The values must also look like timestamps

A matching name alone is not trusted. A **sample of the first 500 rows** is inspected before the column is accepted. Accepted value shapes:

| Shape | Example | Requirement |
|---|---|---|
| Datetime string | `2026-05-27T14:30:00Z` | **ISO-8601 strongly preferred.** At least 90% of sampled values must parse as dates. |
| Unix epoch, **seconds** | `1780000000` | Must be plausible epoch seconds (roughly year 2001–2065). |

Two traps here:

- **Milliseconds, microseconds and nanoseconds are not recognised.** An epoch value like `1780000000000` is out of the accepted range and the column is rejected as a timestamp. Divide to seconds, or convert to ISO-8601.
- **A plain row counter is not a timeline.** A monotonic step index (`0, 1, 2, 3, …`) is *not* accepted as a timestamp by default. If your data has no wall-clock time at all, contact the data team before submitting rather than passing off an index as a time column.

If the name matches but the values do not fit, the guess is **discarded silently** and the run degrades to tabular-only validation. If your data is a time series but the column cannot be made to fit these shapes, declare it as one on submission ([§6.1](#61-declare-the-timestamp-column-explicitly)) so the mismatch is reported rather than passed over.

**Also recommended:** use UTC, with an explicit offset or a trailing `Z`. Local times without an offset are ambiguous across DST boundaries, and mixing offsets within one column produces gap-detection results that are wrong without being obviously wrong.

### 6.4 Regular sampling interval

Gap detection infers an expected sampling interval from the data. Regularly sampled data (a fixed 1 s, 1 min, 5 min cadence) gives clean, meaningful gap reports. Irregular or event-driven data still validates, but every long inter-event interval may be reported as a gap. If your data is event-driven rather than periodically sampled, say so in the metadata `Description` and provenance fields so consumers read the gap report correctly.

---

## 7. Worked examples

The examples below come from a real testbed export (AMF pod resource and latency measurements), to make the conversion concrete.

Original header:

```
time,ram_limit,cpu_limit,ram_usage,cpu_usage,n,mean,lat50,lat75,lat80,lat90,lat95,lat98,lat99,lat100
```

Checked against the rules above:

| Column | Verdict |
|---|---|
| `time` | Unix-epoch **seconds**, monotonically increasing, unique, no nulls. ✅ Passes: `time` is in the recognised name list and the values pass the epoch-seconds check. Renaming to `timestamp` is recommended but not required. |
| `cpu_limit`, `ram_usage`, `cpu_usage`, `n`, `mean`, `lat50`…`lat100` | Clean numeric, no nulls. ✅ Fine as-is. |
| `ram_limit` | Values like `"2048M"`, `"1024M"`, `"4096M"` — a unit suffix baked into the value. ❌ **Fails [§5.2](#52-one-column--one-measured-variable):** the column is inferred as text, so no numeric check can run on it, and it breaks any downstream numeric step. |

### 7.1 Fixing an embedded-unit column

```python
# ram_limit: "2048M"  ->  ram_limit_mb: 2048   (separate the value from the unit)
df["ram_limit_mb"] = df["ram_limit"].str.rstrip("M").astype(int)
df = df.drop(columns=["ram_limit"])
```

The unit does not disappear — it moves into the column name (`ram_limit_mb`) and, ideally, into the unit annotation on that variable's metadata entry, so a downstream consumer does not have to infer "megabytes" from a suffix that the quality checks cannot see anyway.

### 7.2 Pivoting a multi-source (long-format) dataset to wide format

A separate, very common problem: the same measured parameter is reported by **multiple sources** sharing one time axis, distinguished by an ID column. This violates [§5.3](#53-one-row--one-point-in-time-time-series-only) as soon as two sources report at the same instant.

**Before** — long format, one row per `(timestamp, node_id)`; timestamps are not unique:

```
timestamp,            node_id, rsrp, throughput
2026-05-27T10:00:00Z, node-1,  -85,  120.4
2026-05-27T10:00:00Z, node-2,  -91,  98.1
2026-05-27T10:01:00Z, node-1,  -84,  121.0
2026-05-27T10:01:00Z, node-2,  -90,  99.5
```

**After** — wide format, one row per timestamp, source folded into the column name:

```
timestamp,            node_1_rsrp, node_2_rsrp, node_1_throughput, node_2_throughput
2026-05-27T10:00:00Z, -85,         -91,         120.4,             98.1
2026-05-27T10:01:00Z, -84,         -90,         121.0,             99.5
```

Conversion:

```python
import pandas as pd
import re

def snake(name):
    return re.sub(r"_+", "_", re.sub(r"[^0-9a-zA-Z]+", "_", str(name).strip().lower())).strip("_")

long_df = pd.read_csv("raw.csv", parse_dates=["timestamp"])
wide = long_df.pivot(index="timestamp", columns="node_id")
wide.columns = [snake(f"{node_id}_{metric}") for metric, node_id in wide.columns]
wide = wide.sort_index().reset_index()
wide.to_csv("converted.csv", index=False, encoding="utf-8")
```

Notes for anyone doing this conversion:

- **Naming convention: `<source>_<metric>`, already in `snake_case`.** Declare each pivoted column individually in `Dataset columns` — `node_1_rsrp` and `node_2_rsrp` as two distinct variables, rather than one ambiguous `rsrp` that silently meant "whichever source this row happened to be". Applying the normalisation yourself (as `snake()` above does) guarantees the name you declare is the name the pipeline uses, and surfaces any collision at conversion time rather than at validation time. See [§4.2](#42-use-ascii-lowercase-snake_case).
- **Pivoting turns implicit gaps into real nulls**, wherever one source did not report at a timestamp another did. That is expected and correct — it is precisely what imputation exists for — not a sign the pivot went wrong. These gaps will show up as reduced completeness in the quality report; that is the intended outcome, and no reason to avoid the pivot or to pad the cells. See [§5.1](#51-missing-values-are-reported-not-rejected).
- **If sources sample at different cadences**, the pivot still works (it unions all timestamps across sources), but declare the dataset as a time series and name the timestamp column explicitly ([§6.1](#61-declare-the-timestamp-column-explicitly)): the resulting gap pattern is denser than a single-source file and easier to misclassify.
- **This is a different failure mode from [§7.1](#71-fixing-an-embedded-unit-column).** There, a *value* mixed two things (a number and a unit). Here, the *row axis* mixes two things (time and source identity). Both break the same underlying assumption the whole pipeline rests on:

> **one row = one point in time; one column = one measured variable.**

### 7.3 Anti-pattern: many time series stacked in one file

Because this shape is easy to produce and hard to spot, it is worth working through in full. The example below is a network link-monitoring export of the kind testbeds routinely generate:

```
timestamp,link,packet_loss,jitter,outflow_Router,inflow_Router,resolve,avalability,ttl,probe_duration,rtt,LABEL
1750858080,R1-R9,0.0,0.17,4.65,4.6,0.01,1.0,62.0,0.0,0.98,0.0
1750858080,R8-R6,0.0,0.17,4.64,4.64,0.01,1.0,62.0,0.0,1.70,0.0
```

At first glance this looks like a clean, well-formed time series: a proper `timestamp` column in epoch seconds, tidy numeric metrics, a steady 15-second cadence, no stray units, no text in the value columns. It parses without a single error.

It is nevertheless **not one time series — it is many of them interleaved**, one per monitored link, and the pipeline cannot process it as submitted:

| Property | Shape |
|---|---|
| Rows | one per `(timestamp, link)` pair — the number of timestamps multiplied by the number of links |
| Distinct `link` values | many (`R1-R9`, `R8-R6`, `R3-R4`, …) |
| Rows sharing a single timestamp | **as many as there are links** |
| Sampling interval | regular, with every link probed each cycle |

A file of this kind easily runs to hundreds of thousands of rows covering a few days — while the number of genuine points on its timeline is smaller by exactly the number of links.

The `link` column is not a measurement. It is a **series identifier**: it says which of the separately monitored links the row's ten metrics belong to. Every timestamp therefore repeats once per link, which violates [§5.3](#53-one-row--one-point-in-time-time-series-only) on essentially every row of the file.

**What actually happens on submission.** Not a clean rejection, which is what makes this shape worth calling out. The `timestamp` column is detected normally — the name matches and the values are valid epoch seconds — so the dataset is treated as a time series, and then the uniqueness requirement fails across the whole file. Worse is what the values mean if the shape is *not* caught: `rtt` in one row is the round-trip time of `R1-R9` and in the next row of `R8-R6`, so any statistic computed over the column — a mean, a bound, an outlier test, an imputed value — silently mixes every link in the file into one meaningless number. Gap detection is equally confused: it sees a whole cycle of samples where it expects one, and cannot tell a genuinely missing measurement from the normal repetition.

**The fix** is the pivot from [§7.2](#72-pivoting-a-multi-source-long-format-dataset-to-wide-format), with `link` as the discriminator:

```python
import pandas as pd, re

def snake(name):
    return re.sub(r"_+", "_", re.sub(r"[^0-9a-zA-Z]+", "_", str(name).strip().lower())).strip("_")

df = pd.read_csv("link_monitoring_export.csv")
wide = df.pivot(index="timestamp", columns="link")
wide.columns = [snake(f"{link}_{metric}") for metric, link in wide.columns]
wide = wide.sort_index().reset_index()
wide.to_csv("converted.csv", index=False, encoding="utf-8")
```

This yields **one row per timestamp**, and **one column per `(link, metric)` pair** (`r1_r9_rtt`, `r8_r6_jitter`, …) — each an independently meaningful series. The table gets wider by the number of links and shorter by the same factor; no measurement is lost, because the original rows carried exactly these values, merely stacked instead of aligned.

Two further observations on a file of this shape, both worth fixing in the same pass:

- **Column naming** ([§4.2](#42-use-ascii-lowercase-snake_case)). The header mixes cases (`outflow_Router`, `LABEL` alongside `packet_loss`) and carries a typo (`avalability`). Neither blocks validation, but both are worth a pass before submitting: a column name is published into the catalogue and copied into every consumer's code, so a typo becomes permanent vocabulary that is far harder to correct later than now. Proof-reading the header is a two-minute job with a long payoff.
- **Missing values** ([§5.1](#51-missing-values-are-reported-not-rejected)). A substantial share of rows carry a timestamp and a link but no metrics at all — probe cycles that produced no result. That is perfectly acceptable: leave those cells empty, let the completeness figure reflect reality, and use imputation afterwards if the gaps need filling. Do not drop the rows and do not pad them with zeros — a `packet_loss` of `0.0` means *no packets were lost*, which is the opposite of *we did not measure*.

**Splitting into one file per series is the other valid option.** If a wide column per link per metric is unwieldy, submit one file per link instead — each a clean single time series, with the link identity recorded in the metadata rather than in a column. Each is validated as its own distribution ([§3.4](#34-size-and-one-file-per-distribution)). Choose the pivot when the links are analysed together (correlations, network-wide behaviour), and the split when they are analysed independently.

---

### 7.4 What the platform will accept and handle in future

The requirements in this document describe **the current release, not a permanent contract**. 6G-DALI is being developed alongside this first round of contributions, and it is expected to move in two directions at once: accepting a wider range of datasets, and doing more of the preparation work itself. Nothing here is a limitation we are attached to.

#### Widening what can be submitted

The narrowness of [§2](#2-accepted-file-formats) reflects what the validation pipeline reads *today*. Planned extensions:

| Area | Today | Direction of travel |
|---|---|---|
| **File formats** ([§2.2](#22-formats-not-yet-supported)) | CSV, TSV, JSON, JSONL | **Apache Parquet first**, then other columnar and scientific formats (ORC, Feather/Arrow, HDF5) read natively, with no conversion step |
| **Archives and compression** ([§2.1](#21-archives-must-be-expanded-before-upload), [§3.3](#33-compression)) | Expand and decompress before upload | `.zip` / `.tar.gz` / `.csv.gz` accepted directly and unpacked on ingest |
| **Non-tabular data** ([§2.2](#22-formats-not-yet-supported)) | Case-by-case, by arrangement | Defined submission routes for packet captures, raw I/Q samples, images and other non-tabular measurement data, with quality profiles suited to each |
| **Long-format time series** ([§5.3](#53-one-row--one-point-in-time-time-series-only)) | Pivot to wide, or split per series | Long format accepted as a first-class layout, with the series-identifier column declared in the metadata instead |
| **Dataset size** ([§3.4](#34-size-and-one-file-per-distribution)) | 1 GB per upload session | Larger uploads and multi-file datasets handled as a single logical dataset |
| **Dataset kinds** ([§1.1](#11-two-kinds-of-dataset--and-which-rules-apply-to-yours)) | Tabular and time series | Richer support for geospatial, graph/topology and relational multi-table datasets, each with checks appropriate to its shape |

If your dataset falls outside what is accepted today, **tell us rather than reshaping it to fit, or worse, not submitting it** ([Section 9](#9-support)). Knowing that a dataset exists and what shape it is in is more valuable to us than receiving it in a form that cost you a week of conversion — and it is what determines the order of the list above.

#### Automating the preparation

Most of the conversions in this section are mechanical: the pipeline can already *detect* these shapes, and detecting them is most of the work of fixing them. Asking every contributor to do that work by hand is a limitation of the current release, not a design principle.

Planned, in rough order of how reliably each can be done without guessing:

| Preparation step | Today | Planned |
|---|---|---|
| Delimiter and encoding detection ([§3.1](#31-encoding-utf-8-no-byte-order-mark), [§3.2](#32-delimiters-and-decimal-separator)) | You normalise before upload | Detected and handled on ingest, including BOM removal and semicolon/comma-decimal locales |
| Archive expansion ([§2.1](#21-archives-must-be-expanded-before-upload)) | You expand before upload | Archives unpacked automatically, each file registered as its own distribution |
| Additional formats ([§2.2](#22-formats-not-yet-supported)) | You convert to CSV | Parquet and other columnar formats read directly |
| Column-name normalisation ([§4.2](#42-use-ascii-lowercase-snake_case)) | You rename before submission | Normalised on ingest, with the original names preserved in the metadata |
| Row ordering ([§5.3](#53-one-row--one-point-in-time-time-series-only)) | You sort before export | Sorted automatically |
| Long-to-wide pivoting ([§7.2](#72-pivoting-a-multi-source-long-format-dataset-to-wide-format), [§7.3](#73-anti-pattern-many-time-series-stacked-in-one-file)) | You pivot before submission | Series-identifier columns detected and the pivot offered as a proposed transformation for you to confirm |
| Unit extraction ([§7.1](#71-fixing-an-embedded-unit-column)) | You split value from unit | Common unit suffixes detected and proposed as a split, with the unit written into the metadata |

Two of these deserve a caveat. Pivoting and unit extraction change what the data *means*, so they will be offered as a **proposed transformation you confirm**, never applied silently — a derived dataset registered alongside your original, which is never modified ([§5.1](#51-missing-values-are-reported-not-rejected)). The rest are lossless and can safely happen on ingest.

**This round of contributions is what calibrates that roadmap.** The shapes real submissions actually arrive in determine what gets automated first. So if you hit a conversion that feels like it should not be your job, tell us ([Section 9](#9-support)) — that report is genuinely useful, and it is the fastest route to it not being your job next time.

---

## 8. Pre-submission checklist

**File**

- [ ] File is CSV, TSV, JSON or JSONL, with a correct filename extension ([§2](#2-accepted-file-formats)).
- [ ] Encoded as UTF-8, with **no** byte-order mark ([§3.1](#31-encoding-utf-8-no-byte-order-mark)).
- [ ] Delimiter matches the extension (comma for `.csv`, tab for `.tsv`); decimal separator is `.`, with no thousands separators ([§3.2](#32-delimiters-and-decimal-separator)).
- [ ] Not an archive — any `.zip` / `.tar.gz` has been expanded and its data files uploaded individually ([§2.1](#21-archives-must-be-expanded-before-upload)).
- [ ] Source data in Parquet, ORC, Excel or another unsupported format has been converted to CSV ([§2.2](#22-formats-not-yet-supported)).
- [ ] Uncompressed, and within the upload size limit ([§3.3](#33-compression), [§3.4](#34-size-and-one-file-per-distribution)).

**Columns**

- [ ] Header row present; every declared `Dataset columns` name matches the file's header **exactly** ([§4.1](#41-names-must-match-your-declared-metadata-exactly)).
- [ ] Column names are ASCII lowercase `snake_case`, with no two names colliding once punctuation and case are normalised ([§4.2](#42-use-ascii-lowercase-snake_case)).
- [ ] Missing values are left as empty cells, not padded with sentinels (`-1`, `0`, `999`, `"N/A"`); no column is entirely empty ([§5.1](#51-missing-values-are-reported-not-rejected)).
- [ ] No cell mixes a unit, a qualifier or a composite value into the value — units live in the column name or in metadata ([§5.2](#52-one-column--one-measured-variable), [§7.1](#71-fixing-an-embedded-unit-column)).

**Time series** *(skip this block entirely if your dataset has no time axis — see [§1.1](#11-two-kinds-of-dataset--and-which-rules-apply-to-yours))*

- [ ] Dataset declared as a time series on submission, with its timestamp column named — or, failing that, the column is called `timestamp` ([§6.1](#61-declare-the-timestamp-column-explicitly), [§6.2](#62-if-you-dont-it-is-guessed-from-the-header)).
- [ ] Timestamp values are ISO-8601 (UTC preferred) or Unix epoch **seconds** — not milliseconds, not a row counter ([§6.3](#63-the-values-must-also-look-like-timestamps)).
- [ ] Exactly one row per timestamp — verified, e.g. with `df.groupby("timestamp").size().max() == 1` ([§5.3](#53-one-row--one-point-in-time-time-series-only)).
- [ ] No identifier column (`node_id`, `link`, `probe`, …) stacking several series in one file; such data has been pivoted to wide format or split into one file per series ([§7.2](#72-pivoting-a-multi-source-long-format-dataset-to-wide-format), [§7.3](#73-anti-pattern-many-time-series-stacked-in-one-file)).
- [ ] Rows sorted in ascending timestamp order ([§5.3](#53-one-row--one-point-in-time-time-series-only)).
- [ ] Sampling regime (periodic vs. event-driven) described in the metadata, so the gap report is read correctly ([§6.4](#64-regular-sampling-interval)).

---

## 9. Support

Questions about converting a dataset, formats not covered here, or datasets that cannot be made to fit these rules: contact the 6G-DALI data team at **`data@6gdali.eu`** *before* submitting. It is considerably cheaper to agree on a conversion up front than to iterate through failed validation runs.
