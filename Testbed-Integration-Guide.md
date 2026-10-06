# 6G-DALI Testbed Integration Guide

**Version:** 0.1 (draft)
**Project:** 6G-DALI (SNS-JU)
**Audience:** Testbed operators joining the 6G-DALI Data Space, and platform engineers onboarding them
**Companion documents:** [Metadata Application Profile](6G-DALI-Metadata-Application-Profile.md) · [Testbed Metadata JSON Guide](Testbed-Metadata-JSON-guide.md) · [Input Dataset Format](6G-DALI-input-dataset-format.md)

---

## 1. Overview

A testbed joins the 6G-DALI Data Space by running a **testbed EDC connector** next to its own data storage. The testbed keeps ownership of its data. The 6G-DALI Data Space connector pulls data from it under a negotiated contract, so nothing is pushed from the testbed to the platform unprompted.

The testbed's responsibilities are small:

1. Run the connector stack (connector, Postgres, S3-compatible store).
2. Make the connector reachable by the 6G-DALI Data Space connector: over the internet through a TLS reverse proxy (§5).
3. Drop each dataset into its bucket as `<dataset>/metadata.json` plus one or more **CSV (or other format)** data files.

Everything after that is automated. The 6G-DALI Data Space connector transfers the files into the data lake. The central platform then registers each dataset and file in the catalogue and as EDC assets. The testbed's own broker is told when each file has been transferred.

Currently integrated testbeds:

- ISI (Athena RC)
- KUL
- EUR

---

## 2. Architecture

```
          TESTBED SITE                                     6G-DALI CENTRAL
  ┌───────────────────────────────────────┐              ┌────────────────────────────────────────┐
  │                                       │              │                                        │
  │  Testbed operator                     │              │  6G-DALI Data Space (consumer)         │
  │     │ upload <dataset>/metadata.json  │     DSP      │   https://edc.dataspace.6gdali.eu      │
  │     │        <dataset>/*.csv          │              │      │                                 │
  │     ▼                                 │◀────────────▶│      │ 1. catalogue request            │
  │  ┌──────────────┐   ┌──────────────┐  │              │      │ 2. contract negotiation         │
  │  │ S3 store     │◀──│ Testbed EDC  │  │              │      │ 3. PiveauData transfer          │
  │  │              │   │ connector    │  │              │      ▼                                 │
  │  └──────────────┘   │ (provider)   │──┼── files ─────┼─▶ Data Lake                            │
  │                     └──┬────────┬──┘  │              │                                        │
  │                        │        │     │              │   Catalogue ◀──── dataset record       │
  │                        │        └─────┼── metadata ──┼─▶  (6G-DALI catalogue)                 │
  │                        │ transfer     │              │                                        │
  │                        │ completed    │              │                                        │
  │                        ▼              │              │                                        │
  │  ┌──────────────────────────────┐     │              │                                        │
  │  │ RabbitMQ (testbed-internal)  │     │              │                                        │
  │  │ ──▶ testbed's own consumers  │     │              │                                        │
  │  └──────────────────────────────┘     │              │                                        │
  └───────────────────────────────────────┘              └────────────────────────────────────────┘
```

| Component | Role |
|---|---|
| **Testbed connector** | EDC provider built from `ghcr.io/6g-dali/6gdali-testbed-connector`. It exposes the testbed's bucket as an EDC asset and carries 6G-DALI's data-sink extension, which writes the transferred files to the central data lake. It does not talk to the catalogue. |
| **S3-compatible store** | Holds the testbed's datasets. |
| **6G-DALI Data Space connector** | The EDC connector that acts as consumer. It negotiates contracts and receives transfers. |
| **Data Lake** | Central S3-compatible store. Each testbed gets its own bucket, named after `edc.experiment.prefix`. |
| **Central registration** | Central service that watches the data lake. For each new `metadata.json` it registers a dataset in the catalogue, and for each new data file it registers an EDC asset and adds a distribution to the dataset. |
| **6G-DALI catalogue** | DCAT-AP 3.0 catalogue. Datasets are registered here from `metadata.json`. |
| **RabbitMQ (testbed-side)** | A broker run by the testbed. The connector publishes one message per CSV file transferred, so the testbed can see that the transfer completed. It is not connected to the central platform. |

---

## 3. End-to-end flow

### 3.1 One-time setup (per testbed)

1. **Deploy the connector stack** at the testbed (§4).
2. **Make the connector reachable** by the 6G-DALI Data Space connector over the internet (§5).
3. **Register the bucket as an asset** on the testbed connector with the provider preparation script. This creates three things on the connector's management API:
   - an asset with a `6GDaliTestbedExperiments` data address pointing at the bucket,
   - a `no-constraint-policy`,
   - a contract definition that offers every asset under that policy.
4. **Subscribe the 6G-DALI Data Space connector** with the contract preparation script. Acting as consumer, it:
   1. fetches the testbed's offer from its catalogue,
   2. negotiates a contract and waits for `FINALIZED`,
   3. starts a **`PiveauData` PUSH transfer**.

The transfer is long-lived. It stays `STARTED` and keeps polling the bucket, so **steps 3 and 4 run once per deployment, not once per dataset**.

### 3.2 Per-dataset flow (steady state)

1. The operator uploads `metadata.json` **first**, then the CSV files, under one prefix in the bucket (§6).
2. The testbed connector's data plane sees the new objects on its next poll.
3. For each object it transfers the file to the central data lake (bucket `<edc.experiment.prefix>`, under the dataset's UUID directory, see step 4).
4. The data-sink extension only writes the files to the data lake. It does not talk to the catalogue. The bucket is `<edc.experiment.prefix>`. The testbed's directory and file names are not kept: the dataset directory is a random UUID (the first file seen for a source directory creates it, and the mapping is stored in the bucket under `.datasets/`), `metadata.json` keeps its name, and each CSV is stored as `<dataset-uuid>/<file-uuid>.csv`. A file's UUID is remembered (under `.files/`), so sending the same file again overwrites the same object. The original file name is kept as the object's `original-name` metadata.
5. The central registration service sees the new objects on its next poll and handles them:
   - For `metadata.json`, it parses the file and registers a DCAT-AP dataset in the 6G-DALI catalogue (§7). The bucket is the catalogue id and the directory (the dataset's UUID) is the dataset id.
   - For each CSV, it registers an EDC asset and then adds a distribution to the dataset. Per-file fields such as `columns` and `measurement_technique` come from `metadata.json`.
   - It handles `metadata.json` before the CSVs of the same poll. A CSV whose dataset is not registered yet is retried on later polls, so the upload order is no longer critical.
6. After a CSV is written to the data lake, the testbed connector publishes a completion message to the testbed's own RabbitMQ broker (§8).
7. Once registered in the catalogue, the dataset is eligible for central validation and DataOps. This is independent of the testbed's RabbitMQ message. The `dali_dataspace_validate_dataset` Airflow DAG runs format checks, Great Expectations and the DataOps pipeline against it and writes the quality report back into the catalogue as `dqv:QualityMeasurement` entries. Each distribution is retrieved over EDC by its `dali:assetId`.

---

## 4. Deploying the testbed connector

Each testbed connector runs as its own Docker Compose stack:

| Service | Purpose | Notes |
|---|---|---|
| `db` | Postgres 16, EDC persistence | |
| `s3` + `s3-init` | S3-compatible store; `s3-init` creates the bucket on first start | Optional. Omit it if the testbed already has an S3-compatible store, and point the connector at that store instead. |
| `provider` | The EDC connector | Ports 18190–18192 |

### Connector ports

| Port | API | Path |
|---|---|---|
| 18190 | Default / observability | `/api` (health: `/api/check/health`) |
| 18192 | DSP (other connectors connect here) | `/protocol` |

### Key configuration (`<testbed>_connector.properties`)

| Property | Meaning |
|---|---|
| `edc.participant.id` / `edc.ids.id` | Identity of this participant in the data space. Must match what the 6G-DALI Data Space connector expects (e.g. `provider-kul`). |
| `edc.dsp.callback.address` | Address other connectors use to call back. Use the public URL, e.g. `https://edc.<testbed>.6gdali.eu/protocol` (§5). |
| `edc.experiment.prefix` | Prefix for experiment IDs and the central data-lake bucket. |
| `edc.rabbitmq.host` / `.port` / `.username` / `.password` / `.queue` | The testbed's own broker, where transfer-completion messages are published. |
| `edc.catalog.ui.submit.enabled` | Enables dataset submission from the connector's built-in catalogue UI (off by default). |

The connector has **no global storage setting**. Each asset's `dataAddress` carries the endpoint, bucket, credentials and prefix.

### Deploy

```bash
docker compose pull
docker compose up -d
docker compose logs -f provider
curl -s http://localhost:18190/api/check/health
```

Config is mounted as a compose `config`, so changes need `docker compose up -d` (recreate), not just a restart.

---

## 5. Network access

A testbed connector must be reachable by the 6G-DALI Data Space connector over DSP. Publish it on the internet behind a TLS reverse proxy on a dedicated hostname, as the 6G-DALI Data Space connector does (`edc.dataspace.6gdali.eu`). The connector keeps its ports. The proxy exposes them on 443, routed by path.

**Prerequisites**

- A public DNS name for the testbed connector, e.g. `edc.<testbed>.6gdali.eu`, pointing at the host.
- Inbound 443 open to the host.
- A TLS certificate for that name.

**Reverse proxy.** Use the nginx template provided with the connector stack: rename the file, replace the `ISI_EDC_DOMAIN` placeholder, then add the TLS certificate. The template routes by path prefix and passes the path through untouched, because the connector serves each API under its prefix.

| Path | Backend port | Expose publicly? |
|---|---|---|
| `/protocol*` | 18192 (DSP) | **Yes.** This is what the 6G-DALI Data Space connector calls. |
| `/api*` | 18190 (observability, catalogue UI) | Optional. Health checks and the catalogue UI. |

**Connector configuration**

```properties
edc.dsp.callback.address=https://edc.<testbed>.6gdali.eu/protocol
```

**Registering the testbed with the 6G-DALI Data Space connector.** Point the helper scripts at the public URL, with no per-API ports because the proxy routes by path:

```python
PROVIDER_DOMAIN = os.environ.get("<TESTBED>_PROVIDER_DOMAIN", "https://edc.<testbed>.6gdali.eu")
```

Use `https://` throughout.

---

## 6. Preparing and uploading a dataset

### 6.1 Layout in the bucket

```
<bucket>/
└── <dataset-dir>/            e.g. isi-ran-energy-2026-09
    ├── metadata.json         upload this first
    ├── part-1.csv
    └── part-2.csv
```

`<dataset-dir>` becomes part of the experiment ID. When it is not the identifier you want in the catalogue, set `identifier` in `metadata.json`.

### 6.2 Data files

The data files must satisfy the [Input Dataset Format](6G-DALI-input-dataset-format.md). The main rules are:

- UTF-8 with no BOM, comma-delimited, `.` as the decimal separator.
- Header names in ASCII lowercase `snake_case`, matching `columns` in `metadata.json` exactly.
- One row per timestamp for time series. Pivot or split long-format data first.
- No archives and no compression.

### 6.3 Upload

Upload `metadata.json` and the data files to the testbed's bucket, under the dataset directory (§6.1), using your usual S3 tooling. Upload `metadata.json` first.

---

## 7. Dataset metadata (`metadata.json`)

`metadata.json` is the single source for the dataset's catalogue record. The central platform turns it into DCAT-AP 3.0 plus GAIA-X and `dali:` properties.

Mandatory fields: `title`, `description`, `issued`, `license`, `access_rights`, `gdpr_compliant`, `fair_compliant`, `contains_pii`, `sns_project_name`. In practice the connector falls back to defaults for several of these.

A starting `metadata.json` template is provided with the connector stack. The full field reference, including the `testbed_context` block based on the SNS-JU Common Metadata Template (CMT), is in the [Testbed Metadata JSON Guide](Testbed-Metadata-JSON-guide.md).

Rules to remember:

- `issued` and `temporal_*` use `YYYY-MM-DD`.
- `theme`, `language` and `access_rights` take EU authority **codes** (`TECH`, `ENG`, `PUBLIC`), not URLs.
- `produced_by` is your organisation's GAIA-X participant IRI. Ask the platform administrator for it.
- Unknown keys are ignored. Both `snake_case` and `camelCase` are accepted.
- `testbed_context` is optional. Omit unknown fields rather than leaving `TODO` values.
- The file must be named `metadata.json` (case-insensitive) and sit in the dataset's directory. Other `.json` files are not treated as dataset metadata.
- The central platform reads all fields in the Testbed Metadata JSON Guide.

---

## 8. Transfer-completion notifications (testbed-side)

The RabbitMQ connection is **internal to the testbed**. It lets the testbed know when a transfer has completed, for example to trigger its own post-processing or cleanup. It does not feed the central 6G-DALI platform, which does not read this queue. The broker is run and owned by the testbed, and the testbed decides whether to use it.

For every `.csv` file transferred through a `PiveauData` transfer, the testbed connector publishes a message to the configured queue (default exchange, durable).

```json
{"request_id": "isi-ran-energy-2026-09/dataset.csv", "status": "SUCCESS"}
```

| Field | Values | Meaning |
|---|---|---|
| `request_id` | `<dataset dir>/<file>.csv` | The source object key |
| `status` | `SUCCESS` or `FAILED` | `SUCCESS` once the file is in the data lake. `FAILED` if the S3 upload failed and the HTTP fallback was unavailable or also failed. |

Notes:

- `metadata.json` does not produce a notification.
- `SUCCESS` means the file is in the data lake. It does not say anything about the catalogue, which the central platform updates separately.
- The broker host, credentials and queue name are the testbed's choice. The existing stacks use a queue called `ds_connector`.
- If `edc.rabbitmq.host` or `.queue` is unset, notifications are skipped with a warning and the transfer does not fail.

---

## 9. Onboarding checklist for a new testbed

- [ ] Copy a `connector-<testbed>` stack and rename the identifiers (participant ID, bucket, volumes).
- [ ] Set a unique `edc.participant.id`, `edc.ids.id` and `edc.experiment.prefix`.
- [ ] Set `edc.rabbitmq.*` only if the testbed wants completion messages from its own broker.
- [ ] Make the connector reachable (§5): create the DNS record, install the nginx vhost with TLS, expose only `/protocol` (and optionally `/api`), and set `edc.dsp.callback.address` to the `https://` URL.
- [ ] `docker compose up -d` and check `/api/check/health`.
- [ ] Create an S3 access key for the connector and register the bucket with `0-prepare_provider_<testbed>_dali.py`.
- [ ] Run `1-prepare-contract_<testbed>_dali.py` to start the transfer.
- [ ] Obtain your `produced_by` participant IRI from the platform administrator.
- [ ] Upload a small test dataset (`metadata.json`, then a CSV) and check that:
  - the dataset appears in the 6G-DALI catalogue,
  - the file lands in the central data lake,
  - a `SUCCESS` message arrives on the testbed's own queue (if RabbitMQ is configured).
