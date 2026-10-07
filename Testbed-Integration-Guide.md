# 6G-DALI Testbed Integration Guide

**Version:** 0.2 (draft)
**Project:** 6G-DALI (SNS-JU)
**Audience:** Testbed operators joining the 6G-DALI Data Space
**Companion documents:** [Metadata Application Profile](6G-DALI-Metadata-Application-Profile.md) · [Testbed Metadata JSON Guide](Testbed-Metadata-JSON-guide.md) · [Input Dataset Format](6G-DALI-input-dataset-format.md)

---

## 1. Overview

A testbed joins the 6G-DALI Data Space by running a **testbed EDC connector** next to its own data storage. The testbed keeps ownership of its data. The 6G-DALI Data Space connector pulls data from it under a negotiated contract, so nothing is pushed from the testbed to the platform unprompted.

Two roles take part:

- The **testbed operator** runs the connector and uploads datasets.
- The **platform administrator** registers the testbed in the DataOps UI, which prepares its place in the platform, and then connects it to the data space.

The testbed operator's responsibilities are small:

1. Deploy the connector stack from the bundle the administrator provides (connector, Postgres, S3-compatible store) (§4).
2. Make the connector reachable by the 6G-DALI Data Space connector: over the internet through a TLS reverse proxy (§5).
3. Register the testbed's dataset bucket as an asset on the connector, using the connector's asset page (§4.3).
4. Drop each dataset into its bucket as `<dataset>/metadata.json` plus one or more **CSV (or other format)** data files.

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
| **Testbed connector** | EDC provider built from `ghcr.io/6g-dali/6gdali-testbed-connector`. It exposes the testbed's bucket as an EDC asset and carries 6G-DALI's data-sink extension, which writes the transferred files to the central data lake. It also serves the asset page where the operator registers the bucket (§4.3). It does not talk to the catalogue. |
| **S3-compatible store** | Holds the testbed's datasets. |
| **6G-DALI Data Space connector** | The EDC connector that acts as consumer. It negotiates contracts and receives transfers. |
| **DataOps UI and orchestrator** | The administrators' control plane. The orchestrator keeps the testbed registry in its own Postgres database, provisions each testbed's bucket, Data Lake key and catalogue, generates the connector bundle, and uses the Data Space connector to find the testbed's asset, negotiate the contract and start the transfer. |
| **Data Lake** | Central S3-compatible store (MinIO). Each testbed gets its own bucket, named after `edc.experiment.prefix`, and its own access key that can only list, read and write that bucket. |
| **Central registration** | Central service (the s3-asset-monitor) that watches the data lake. For each new `metadata.json` it registers a dataset in the catalogue, and for each new data file it registers an EDC asset and adds a distribution to the dataset. |
| **6G-DALI catalogue** | DCAT-AP 3.0 catalogue. Datasets are registered here from `metadata.json`. Each testbed has its own catalogue, named after its bucket. |
| **RabbitMQ (testbed-side)** | A broker run by the testbed. The connector publishes one message per CSV file transferred, so the testbed can see that the transfer completed. It is not connected to the central platform. |

---

## 3. End-to-end flow

### 3.1 Onboarding a testbed (once per testbed)

The **platform administrator**, in the DataOps UI (Testbeds):

1. **Registers the testbed.** The registry assigns its identity: participant ID, experiment prefix, bucket and DSP URL.
2. **Provisions it.** This creates the testbed's Data Lake bucket, a bucket-scoped Data Lake key and its catalogue.
3. **Downloads the connector bundle** and gives it to the operator. The bundle holds the connector properties with the assigned identity, a docker-compose stack and an nginx vhost (§4.1).

The **testbed operator**:

4. **Deploys the connector stack** from the bundle (§4) and **makes it reachable** by the 6G-DALI Data Space connector over the internet (§5).
5. **Registers the bucket as an asset** on the connector, on its asset page (§4.3). This creates three things on the connector: an asset with a `6GDaliTestbedExperiments` data address pointing at the bucket, a `no-constraint-policy`, and a contract definition that offers every asset under that policy.

The **platform administrator** then connects it, from the testbed's page in the DataOps UI:

6. **Find asset.** The Data Space connector sends a catalogue request to the testbed's connector and the registry stores what it offers. A successful reply also proves the connector is reachable.
7. **Negotiate contract.** The Data Space connector, acting as consumer, negotiates a contract on the asset's offer and waits for `FINALIZED`.
8. **Start transfer.** The Data Space connector starts a **`PiveauData` PUSH transfer**, using the testbed's own Data Lake key.

The transfer is long-lived. It stays `STARTED` and keeps polling the bucket, so **steps 6 to 8 run once per asset, not once per dataset**.

The Data Lake key never reaches the testbed. The Data Space connector supplies it, in the destination of the transfer request, and the testbed's data plane uses it only to write into the testbed's own bucket. For this reason the connector properties contain no Data Lake credentials.

The preparation scripts used before (`0-prepare_provider_*` and `1-prepare-contract_*`) are no longer needed for a new testbed: the asset page and the DataOps UI replace them.

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

### 4.1 The connector bundle

The administrator downloads the bundle from the testbed's page in the DataOps UI. It is a zip with:

| File | Purpose |
|---|---|
| `<testbed>_connector.properties` | The connector configuration, with the identity assigned by the registry. Do not change the identity values: the Data Space connector expects exactly these. |
| `docker-compose.yaml` | The stack below. |
| `nginx/<domain>.conf` | A reverse-proxy vhost for the connector (§5). |
| `template/metadata.json` | A starting point for a dataset's metadata (§7). |
| `README.md` | The identity and the steps for the operator. |

The passwords of the testbed's own Postgres and S3 store are generated into the bundle. The same testbed always gets the same bundle contents, so downloading it again does not change them.

### 4.2 The stack

Each testbed connector runs as its own Docker Compose stack:

| Service | Purpose | Notes |
|---|---|---|
| `db` | Postgres 16, EDC persistence | |
| `rustfs` + `rustfs-init` | S3-compatible store; `rustfs-init` creates the bucket on first start | Optional. Omit them if the testbed already has an S3-compatible store, and point the asset at that store instead. |
| `provider` | The EDC connector | Ports 18190–18192 |

#### Connector ports

| Port | API | Path |
|---|---|---|
| 18190 | Default / observability, and the connector's catalogue and asset pages | `/api` (health: `/api/check/health`) |
| 18191 | Management (not published to the internet) | `/management` |
| 18192 | DSP (other connectors connect here) | `/protocol` |

#### Key configuration (`<testbed>_connector.properties`)

| Property | Meaning |
|---|---|
| `edc.participant.id` / `edc.ids.id` | Identity of this participant in the data space, assigned by the registry (for example `provider-kul`). Do not change them. |
| `edc.dsp.callback.address` | Address other connectors use to call back. Use the URL the Data Space connector can reach, for example `https://edc.<testbed>.6gdali.eu/protocol` (§5). |
| `edc.experiment.prefix` | Prefix for experiment IDs and the central data-lake bucket, assigned by the registry. |
| `edc.catalog.ui.asset.admin.key` | The key that unlocks the asset page (§4.3). Without it the page is disabled. |
| `edc.rabbitmq.host` / `.port` / `.username` / `.password` / `.queue` | The testbed's own broker, where transfer-completion messages are published. Optional. |
| `edc.catalog.ui.submit.enabled` | Enables dataset submission from the connector's built-in catalogue UI (off by default). |

The connector has **no global storage setting**. Each asset's `dataAddress` carries the endpoint, bucket, credentials and prefix.

#### Deploy

```bash
docker compose pull
docker compose up -d
docker compose logs -f provider
curl -s http://localhost:18190/api/check/health
```

Config is mounted as a compose `config`, so changes need `docker compose up -d` (recreate), not just a restart.

### 4.3 Registering the dataset asset

The connector's management API is not published, so the operator registers the bucket on the connector's own asset page:

1. Open `https://<connector-domain>/api/catalog/register-asset` (or `http://localhost:18190/api/catalog/register-asset` on the host).
2. Enter the **admin key**: the value of `edc.catalog.ui.asset.admin.key` in the properties file. The page answers 404 when no key is configured.
3. Fill in the asset:

| Field | Value |
|---|---|
| Asset id | The id the platform administrator gave you, if any. |
| S3 endpoint | The store, as seen from the connector, for example `http://rustfs:9000`. |
| Bucket | The bucket holding the datasets. |
| Access key and secret key | An access key of the testbed's own S3 store (created in its console). This is not the Data Lake key. |
| Prefix | Optional. Leave it empty to watch the whole bucket. |

4. Register. In one step this creates the asset, the `no-constraint-policy` and a contract definition that offers every asset under it.

The asset's data-address type is `6GDaliTestbedExperiments`. The page lists the assets of that type and says how many assets of other types it hides. An asset of another or an older type (for example `MinioFiles`) cannot be transferred by the connector's data plane: remove it and register it again.

Until the asset exists, the testbed offers nothing to the data space. Tell the platform administrator when it is registered.

---

## 5. Network access

A testbed connector must be reachable by the 6G-DALI Data Space connector over DSP. Publish it on the internet behind a TLS reverse proxy on a dedicated hostname, as the 6G-DALI Data Space connector does (`edc.dataspace.6gdali.eu`). The connector keeps its ports. The proxy exposes them on 443, routed by path.

**Prerequisites**

- A public DNS name for the testbed connector, e.g. `edc.<testbed>.6gdali.eu`, pointing at the host.
- Inbound 443 open to the host.
- A TLS certificate for that name.

**Reverse proxy.** Use the nginx vhost from the connector bundle (`nginx/<domain>.conf`) and add the TLS certificate. It routes by path prefix and passes the path through untouched, because the connector serves each API under its prefix. The management API (18191) and the control API (18193) are deliberately not published.

| Path | Backend port | Expose publicly? |
|---|---|---|
| `/protocol*` | 18192 (DSP) | **Yes.** This is what the 6G-DALI Data Space connector calls. |
| `/api*` | 18190 (observability, catalogue and asset pages) | Needed for the asset page (§4.3), which is protected by its admin key. Otherwise optional. |

**Connector configuration**

```properties
edc.dsp.callback.address=https://edc.<testbed>.6gdali.eu/protocol
```

Use `https://` throughout.

**Testbeds on a private network.** A testbed that is only reachable over a private network (a tailnet, for example) registers the address the Data Space connector can reach, such as `http://<host>:18192/protocol`. What matters is that the DSP URL registered in the DataOps UI is exactly the address the Data Space connector uses to reach the testbed's connector.

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

A starting `metadata.json` template is provided in the connector bundle. The full field reference, including the `testbed_context` block based on the SNS-JU Common Metadata Template (CMT), is in the [Testbed Metadata JSON Guide](Testbed-Metadata-JSON-guide.md).

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

## 9. Onboarding checklist for a testbed operator

- [ ] Deploy the stack from the bundle: `docker compose up -d` and check `/api/check/health` (§4).
- [ ] Make the connector reachable (§5): create the DNS record, install the nginx vhost with TLS, expose `/protocol` (and `/api` for the asset page), and make sure `edc.dsp.callback.address` is the `https://` URL.
- [ ] Create an access key in the testbed's own S3 store for the connector.
- [ ] Register the bucket as an asset on the connector's asset page (§4.3) and tell the administrator.
- [ ] Set `edc.rabbitmq.*` only if the testbed wants completion messages from its own broker.
- [ ] Upload a small test dataset (`metadata.json`, then a CSV) and check that:
  - the dataset appears in the 6G-DALI catalogue,
  - the file lands in the central data lake,
  - a `SUCCESS` message arrives on the testbed's own queue (if RabbitMQ is configured).
