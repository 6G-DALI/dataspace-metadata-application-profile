# 6G-DALI Dataset Metadata File Guide

**Version:** 1.0  
**For:** Testbed owners preparing dataset metadata for the 6G-DALI Data Space  
**Platform:** piveau-hub (DCAT-AP 3.0 compliant)

---

## Overview

When uploading a dataset to the 6G-DALI Data Space, you must provide a `metadata.json` file alongside the data files. This file describes your dataset and is automatically transformed into a DCAT-AP / GAIA-X compliant RDF record registered in the piveau-hub catalogue.

The `metadata.json` file must be sent **before or alongside** the CSV data files in the same EDC transfer, with the `.json` extension.

---

## Field Reference

### Obligation levels

| Symbol | Meaning |
|--------|---------|
| **M** | Mandatory — the field must be present |
| **R** | Recommended — strongly encouraged for discoverability |
| **O** | Optional — include if applicable |

---

### Core Identity

| Field | Obligation | Type | Description |
|-------|-----------|------|-------------|
| `title` | **M** | string | Human-readable name of the dataset |
| `description` | **M** | string | Free-text description of the dataset content |
| `identifier` | O | string | Explicit unique identifier (UUID or DOI). If omitted, the dataset directory name is used |
| `issued` | **M** | string | Date first published. Format: `YYYY-MM-DD` |
| `version` | O | string | Version label (e.g. `"1.0"`) |

> **Note:** `issued` must be in `YYYY-MM-DD` format. Datetime strings with timestamps are not accepted.  
> `dct:modified` is automatically set to the same value as `issued` on initial registration.

---

### Rights and License

| Field | Obligation | Type | Default | Description |
|-------|-----------|------|---------|-------------|
| `license` | **M** | string (URI) | `https://creativecommons.org/licenses/by/4.0/` | License URI. Use a standard license URI (see examples below) |
| `access_rights` | **M** | string | `PUBLIC` | Access condition code from the EU Access Right vocabulary. Values: `PUBLIC`, `RESTRICTED`, `NON_PUBLIC` |

**Common license URIs:**

| License | URI |
|---------|-----|
| CC BY 4.0 | `https://creativecommons.org/licenses/by/4.0/` |
| CC BY-SA 4.0 | `https://creativecommons.org/licenses/by-sa/4.0/` |
| CC BY-NC 4.0 | `https://creativecommons.org/licenses/by-nc/4.0/` |
| MIT | `https://opensource.org/licenses/MIT` |
| Apache 2.0 | `https://opensource.org/licenses/Apache-2.0` |

---

### Classification

| Field | Obligation | Type | Default | Description |
|-------|-----------|------|---------|-------------|
| `theme` | R | string | `TECH` | EU Data Theme code. Most 5G/6G datasets use `TECH` |
| `keywords` | R | array of strings | — | Free keyword tags for discoverability |
| `language` | O | string | `ENG` | Language code from the EU MDR Languages vocabulary (e.g. `ENG`, `GRC`, `DEU`) |

---

### Agents

| Field | Obligation | Type | Description |
|-------|-----------|------|-------------|
| `publisher` | R | string | Name of the organisation publishing the dataset |
| `creator_name` | R | string | Full name of the person or team that created the dataset |
| `creator_email` | R | string | E-mail address of the creator |
| `contact_email` | R | string | Contact e-mail for dataset enquiries |
| `produced_by` | R | string (URI) | URI of your organisation as a GAIA-X participant in the 6G-DALI Data Space. Contact the platform administrator for your URI |

---

### Project Context

| Field | Obligation | Type | Default | Description |
|-------|-----------|------|---------|-------------|
| `sns_project_name` | **M** | string | `6G-DALI` | SNS-JU project name. Leave as default unless instructed otherwise |

---

### Content Description

| Field | Obligation | Type | Description |
|-------|-----------|------|-------------|
| `columns` | R | array of strings | List of column/variable names in the dataset |
| `measurement_technique` | R | string | Description of how measurements were collected (e.g. `"Prometheus-based 5G NR KPI monitoring with 1s scrape interval"`) |
| `record_count` | O | integer | Approximate number of rows/records |
| `file_format` | O | string | File format (e.g. `"CSV"`, `"JSON"`) |
| `number_of_files` | O | integer | Number of data files in this dataset |

---

### 5G/6G Testbed Context (`testbed_context`)

The `testbed_context` object captures your testbed infrastructure and experimentation parameters following the **SNS-JU Common Metadata Template (CMT) V1.0**. All sub-fields are optional — provide only those that apply to your setup.

#### Infrastructure (CMT Group 3A)

| Field | Type | Accepted values / example |
|-------|------|--------------------------|
| `underlay_platform` | string (URI) | URI of your testbed platform |
| `environment` | string | `indoors` · `urban` · `rural` · `mixed` |
| `network_domain` | string | `RAN` · `Transport` · `CORE` · `E2E` |
| `ran_3gpp_release` | string | `"Release 15"` · `"Release 16"` · `"Release 17"` · `"Release 18"` |
| `ran_nr_type` | string | `NR-SA` · `NR-NSA` · `LTE` |
| `ran_split` | string | `DU-RU split` · `No-Split` · `CU-DU split` |
| `ran_focused_technology` | string | `O-RAN` · `JSAC` · `RIC` · `No_focus` |
| `ran_coverage_type` | string | `Single_Macro` · `Single_Micro` · `Multicell_setup` |
| `ran_frequency_band` | string | `"n78"` · `"n77"` · `"n28"` · `"B3"` etc. |
| `ran_bandwidth_mhz` | integer | `20` · `50` · `100` |
| `ran_max_end_devices` | integer | e.g. `10` |
| `ran_mobility_model` | string | `static` · `pedestrian` · `vehicular` · `UAV` |
| `core_release` | string | `"Release 17"` |
| `core_solution` | string | `OpenSource` · `Commercial` |
| `transport_type` | string | `wired` · `microwave` · `fiber_optics` · `satellite` |
| `compute_orchestrator_type` | string | `Kubernetes` · `OpenStack` · `OSM` · `ONAP` |
| `compute_gpu_use` | boolean | `true` · `false` |
| `compute_virtualization_type` | string | `KVM` · `Docker` · `Bare-metal` |
| `compute_infrastructure_type` | string | `"private-edge node"` · `"public cloud"` · `"HPC cluster"` |

#### Service Description (CMT Group 3B)

| Field | Type | Accepted values |
|-------|------|----------------|
| `traffic_origin` | string | `Manual` · `Application` |
| `traffic_pattern` | string | `"UL/DL UDP"` · `"DL TCP"` · `"UL UDP"` etc. |
| `slice_type` | string | `slice101` · `Multi-slice` · `No_slicing` |
| `reference_plane` | string | `control plane` · `data plane` · `management plane` |
| `related_vertical` | string | `Vertical_agnostic` · `CAM` · `HEALTH` · `INDUSTRY` · `MEDIA` |

#### Experimentation & Metrics (CMT Group 3C)

| Field | Type | Accepted values |
|-------|------|----------------|
| `observation_point_horizontal` | string | `E2E Application layer` · `End device to Access` · `DU to CU` · `Access to Edge` · `Access to Core` · `Core to Cloud` |
| `observation_point_vertical` | string | `Radio Level` · `Network Layer` · `Application Layer` · `Compute Resource-level` · `cross-layer` |
| `measurement_family` | array of strings | 3GPP TS 28.552 families: `"DRB"` · `"RRC"` · `"RRU"` · `"L1M"` · `"PEE"` · `"QF"` etc. |
| `measurement_tool` | array of strings | e.g. `"Prometheus exporter"` · `"tcpdump"` · `"iperf3"` |

---

## Complete Example

```json
{
  "title": "5G RAN KPI Measurements — Urban Scenario",
  "description": "5G NR QoS measurement dataset collected in an urban macro-cell environment. Contains per-gNB KPI time-series exported via Prometheus.",
  "issued": "2025-11-24",
  "version": "1.0",

  "license": "https://creativecommons.org/licenses/by/4.0/",
  "access_rights": "PUBLIC",

  "publisher": "EURECOM",
  "creator_name": "EURECOM 5G Lab",
  "creator_email": "5glab@eurecom.fr",
  "contact_email": "5glab@eurecom.fr",
  "produced_by": "https://dali-project.eu/participant/eurecom",

  "sns_project_name": "6G-DALI",

  "theme": "TECH",
  "keywords": ["5G", "RAN", "KPI", "gNB", "QoS"],

  "columns": [
    "source",
    "metric_id",
    "timestamp",
    "unit",
    "value",
    "IP_address",
    "gNB_id"
  ],
  "measurement_technique": "Prometheus-based 5G NR KPI monitoring with 1s scrape interval",
  "record_count": 3239,
  "file_format": "CSV",
  "number_of_files": 1,

  "testbed_context": {
    "environment": "urban",
    "network_domain": "RAN",
    "ran_3gpp_release": "Release 17",
    "ran_nr_type": "NR-SA",
    "ran_frequency_band": "n78",
    "ran_bandwidth_mhz": 100,
    "ran_max_end_devices": 10,
    "ran_mobility_model": "static",
    "compute_orchestrator_type": "Kubernetes",
    "compute_gpu_use": false,
    "compute_virtualization_type": "Docker",
    "traffic_origin": "Application",
    "reference_plane": "data plane",
    "observation_point_vertical": "Network Layer",
    "measurement_family": ["DRB", "RRU"],
    "measurement_tool": ["Prometheus exporter"]
  }
}
```

---

## Minimal Example (mandatory fields only)

```json
{
  "title": "My Dataset",
  "description": "Brief description of the dataset.",
  "issued": "2025-11-24",
  "license": "https://creativecommons.org/licenses/by/4.0/",

  "access_rights": "PUBLIC",
  "sns_project_name": "6G-DALI"
}
```

---

## Notes

- **File name**: name the file `metadata.json` and place it in the experiment directory alongside your CSV files.
- **Date format**: `issued` must be `YYYY-MM-DD`. Do not include time or timezone. `dct:modified` is set automatically to the same value as `issued` on first registration.
- **`produced_by`**: contact the 6G-DALI platform administrator to obtain the correct URI for your organisation.
- **`testbed_context`**: the entire object is optional. If provided, include only the fields you have values for — unknown or missing fields are simply omitted from the catalogue record.
- **`keywords`**: use English language tags wherever possible to maximise discoverability across the federated catalogue.