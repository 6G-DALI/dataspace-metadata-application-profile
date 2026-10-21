# 6G-DALI Testbed Integration Guide

**Version:** 0.3 (draft)
**Project:** 6G-DALI (SNS-JU)
**Audience:** Testbed operators joining the 6G-DALI Data Space, and the platform administrators who onboard them
**Companion documents:** [Metadata Application Profile](6G-DALI-Metadata-Application-Profile.md) · [Testbed Metadata JSON Guide](Testbed-Metadata-JSON-guide.md) · [Input Dataset Format](6G-DALI-input-dataset-format.md)

---

## 1. Overview

A testbed joins the 6G-DALI Data Space in one of two ways, chosen when it is registered:

- **With its own connector.** The testbed runs a **testbed EDC connector** next to its own data storage and keeps its data. The 6G-DALI Data Space connector pulls the data under a negotiated contract.
- **Hosted by the facility connector.** The testbed runs nothing. It gets an **upload bucket** in the 6G-DALI Data Lake and a key for it, and uploads its datasets there. The 6G-DALI facility connector offers that bucket and moves each uploaded file into the Data Space.

Either way, every dataset ends up in the testbed's own **catalogue** in the 6G-DALI catalogue, with the visibility its `metadata.json` asks for in `access_rights` (**Members only** when it asks for none). Administrators can change it afterwards (§9).

Two roles take part:

- The **testbed operator** runs the connector (or uses the upload kit) and uploads datasets.
- The **platform administrator** registers the testbed in the DataOps UI, which prepares its place in the platform, and connects it to the data space.

The testbed operator's responsibilities:

| With its own connector | Hosted by the facility connector |
|---|---|
| Deploy the connector stack from the bundle the administrator provides (§4). | Receive the upload kit from the administrator (§6). |
| Make the connector reachable by the 6G-DALI Data Space connector, through a TLS reverse proxy (§5). | — |
| Register the dataset bucket as an asset on the connector's catalogue page (§4.3). | — |
| Drop each dataset into the bucket as `<dataset>/metadata.json` plus data files (§7). | Upload each dataset to the upload bucket the same way (§7). |

Everything after that is automated. The files are transferred into the Data Lake, and the central platform registers each dataset and file in the catalogue and as EDC assets.

Currently integrated testbeds: ISI (Athena RC), KUL, EUR (staging catalogues), NITOS, Netcloud and 5GC.

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
  │  └──────────────┘   │ (provider)   │──┼── files ─────┼─▶ Data Lake ──▶ asset monitor          │
  │                     └──┬───────────┘  │              │                     │                  │
  │                        │ transfer     │              │                     ▼                  │
  │                        │ completed    │              │   Catalogue (the testbed's catalogue)  │
  │                        ▼              │              │                                        │
  │  ┌──────────────────────────────┐     │              │                                        │
  │  │ RabbitMQ (testbed-internal)  │     │              │                                        │
  │  └──────────────────────────────┘     │              │                                        │
  └───────────────────────────────────────┘              └────────────────────────────────────────┘
```

A **hosted** testbed has no site-side part. It uploads straight into its upload bucket in the Data Lake. The facility connector is provider and consumer at once: it offers the upload bucket as an asset, negotiates with itself and transfers the files into the testbed's Data Lake bucket. From there the flow is the same.

| Component | Role |
|---|---|
| **Testbed connector** | EDC provider built from `ghcr.io/6g-dali/6gdali-testbed-connector`. It exposes the testbed's bucket as an EDC asset and carries 6G-DALI's data-sink extension, which writes the transferred files to the central Data Lake. It also serves the catalogue page, whose Testbed asset panel is where the operator registers the bucket's asset (§4.3). It does not talk to the catalogue. Not used by hosted testbeds. |
| **S3-compatible store** | Holds the testbed's datasets: the testbed's own store, or, for a hosted testbed, its upload bucket in the Data Lake. |
| **6G-DALI Data Space (facility) connector** | The central EDC connector. It acts as consumer for every testbed, and as provider for hosted testbeds. |
| **DataOps UI and orchestrator** | The administrators' control plane. The orchestrator keeps the testbed registry, provisions each testbed (Data Lake bucket and key, catalogue, Keycloak group, and for hosted testbeds the upload bucket, upload key, asset and transfer), generates the connector bundle, and drives the Data Space connector. The testbed's page also manages its datasets and who can access them. |
| **Data Lake** | Central S3-compatible store (MinIO). Each testbed gets its own bucket, named after `edc.experiment.prefix`, and an access key that can only list, read and write that bucket. A hosted testbed also gets `<bucket>-upload`. |
| **Central registration (asset monitor)** | Watches the testbeds' Data Lake buckets. For each new `metadata.json` it registers a dataset in the catalogue, and for each new data file it registers an EDC asset and adds a distribution to the dataset. Its status page lets an administrator re-register files it skipped (§3.3). |
| **6G-DALI catalogue** | DCAT-AP 3.0 catalogue (piveau, with access control). Each testbed has its own catalogue, named after its bucket, public or private (§9). The public front end is `https://catalogue.dataspace.6gdali.eu`. |
| **RabbitMQ (testbed-side)** | A broker run by the testbed. Its connector publishes one message per CSV file transferred. It is not connected to the central platform. Testbeds with their own connector only. |

---

## 3. End-to-end flow

### 3.1 Onboarding a testbed with its own connector

The **platform administrator**, in the DataOps UI (Testbeds → Register testbed):

1. **Registers the testbed**, choosing *Its own connector* and whether its catalogue is **public or private** (§9; fixed once the catalogue exists). The registry assigns its identity: participant ID, experiment prefix, bucket and DSP URL.
2. **Provisions it.** This creates the testbed's Data Lake bucket, a bucket-scoped Data Lake key, its catalogue and its Keycloak group.
3. **Downloads the connector bundle** (testbed page → Overview) and gives it to the operator (§4.1).

The **testbed operator**:

4. **Deploys the connector stack** from the bundle (§4) and **makes it reachable** by the Data Space connector (§5).
5. **Registers the bucket as an asset** on the connector, in the Testbed asset panel of its catalogue page (§4.3).

The **platform administrator** then connects it, on the testbed page's **Assets** tab:

6. **Find asset.** The Data Space connector sends a catalogue request to the testbed's connector, and the registry stores what it offers. A successful reply also proves the connector is reachable.
7. **Negotiate contract.** The Data Space connector negotiates a contract on the asset's offer and waits for `FINALIZED`.
8. **Start transfer.** The Data Space connector starts a **`PiveauData` PUSH transfer**, using the testbed's own Data Lake key.

The transfer is long-lived. It stays `STARTED` and keeps polling the bucket, so **steps 6 to 8 run once per asset, not once per dataset**.

The Data Lake key never reaches the testbed. The Data Space connector supplies it in the transfer request, and the testbed's data plane uses it only to write into the testbed's own bucket. The connector properties therefore contain no Data Lake credentials.

### 3.2 Onboarding a hosted testbed

The **platform administrator**, in the DataOps UI:

1. **Registers the testbed**, choosing *Hosted by the facility connector* and public or private (§9).
2. **Provisions it.** Besides the bucket, key, catalogue and group, this creates the **upload bucket** `<bucket>-upload` and an **upload key** limited to it. It also registers the upload bucket as an asset on the facility connector, and **starts the transfer**: the facility connector negotiates with itself and transfers into the testbed's Data Lake bucket.
3. **Hands over the upload kit** shown at the end of the wizard (§6): endpoint, bucket, access key and **secret key**. The secret is shown only this once and is not stored. If it is lost, *New upload key* on the testbed's Overview replaces the key.

The **testbed operator** then uploads datasets (§7). There is nothing to deploy.

The transfer is started during provisioning on purpose: it only picks up files that arrive while it runs, so it must be running before the first upload. If the facility connector restarts, files uploaded while it is down are not transferred. The **Assets** tab shows whether the transfer is active (*Find transfer*) and restarts it (*Start transfer*).

### 3.3 Per-dataset flow (steady state)

1. The operator uploads `metadata.json` **first**, then the data files, under one directory in the bucket (§7).
2. The connector's data plane (the testbed's, or the facility connector's for a hosted testbed) sees the new objects on its next poll.
3. It transfers each file to the central Data Lake, into the testbed's bucket. The testbed's directory and file names are not kept: the dataset directory becomes a random UUID (the mapping is stored in the bucket under `.datasets/`), `metadata.json` keeps its name, and each CSV is stored as `<dataset-uuid>/<file-uuid>.csv`. A file's UUID is remembered (under `.files/<dataset-dir>/<file name>`), so sending the same file again overwrites the same object. The catalogue lists each file under its UUID name.
4. The central registration service sees the new objects on its next poll:
   - For `metadata.json`, it registers a DCAT-AP dataset in the testbed's catalogue (§8). The bucket is the catalogue id, the directory (the dataset's UUID) the dataset id. A new dataset gets the visibility its `access_rights` asks for, **Members only** without one (§9).
   - For each CSV, it registers an EDC asset and adds a distribution to the dataset, described from the file itself: its columns from its header line, its format, size and checksum. `measurement_technique` and `license` come from `metadata.json`.
   - It handles `metadata.json` before the CSVs of the same poll. A CSV whose dataset is not registered yet is retried on later polls, so the upload order is not critical.
5. With its own connector, the testbed's connector also publishes a completion message to the testbed's RabbitMQ broker (§10).
6. Once registered, the dataset is eligible for central validation and DataOps: the `dali_dataspace_validate_dataset` Airflow DAG runs format checks, Great Expectations and the DataOps pipeline, and writes the quality report back into the catalogue as `dqv:QualityMeasurement` entries. Each distribution is retrieved over EDC by its `dali:assetId`.
7. An administrator can **change the dataset's visibility** later, on the DataOps edit page (§9).

**Files the registration missed.** The registration service only handles files that appear while it runs; files already in a bucket when it starts are marked as seen and skipped. Its status page (central, for administrators) lists every dataset and file per catalogue, whether each is in the catalogue, and re-registers the missing ones on request. Re-registering never adds a file twice and never changes a dataset's visibility.

---

## 4. Deploying the testbed connector

This section applies to testbeds with their own connector.

### 4.1 The connector bundle

The administrator downloads the bundle from the testbed's page in the DataOps UI (Overview). It is a zip with:

| File | Purpose |
|---|---|
| `<testbed>_connector.properties` | The connector configuration, with the identity assigned by the registry. Do not change the identity values: the Data Space connector expects exactly these. |
| `docker-compose.yaml` | The stack below. |
| `nginx/<domain>.conf` | A reverse-proxy vhost for the connector (§5). |
| `template/metadata.json` | A starting point for a dataset's metadata (§8). |
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
| 18190 | Default / observability, and the connector's catalogue page | `/api` (health: `/api/check/health`) |
| 18191 | Management (not published to the internet) | `/management` |
| 18192 | DSP (other connectors connect here) | `/protocol` |

#### Key configuration (`<testbed>_connector.properties`)

| Property | Meaning |
|---|---|
| `edc.participant.id` / `edc.ids.id` | Identity of this participant in the data space, assigned by the registry (for example `provider-kul`). Do not change them. |
| `edc.dsp.callback.address` | Address other connectors use to call back. Use the URL the Data Space connector can reach, for example `https://edc.<testbed>.6gdali.eu/protocol` (§5). |
| `edc.experiment.prefix` | Prefix for experiment IDs and the central Data Lake bucket, assigned by the registry. |
| `edc.catalog.ui.asset.admin.key` | The key that unlocks the Testbed asset panel on the catalogue page (§4.3). Without it the panel is not shown. |
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

### 4.3 Registering the testbed asset

The connector's management API is not published, so the operator manages the testbed's asset on the connector's own catalogue page. A testbed has a **single asset**: the dataset bucket it offers to the data space.

1. Open the catalogue page, `https://<connector-domain>/api/catalog` (or `http://localhost:18190/api/catalog` on the host), and click the **Testbed asset** card. The card is only shown when `edc.catalog.ui.asset.admin.key` is set in the properties file.
2. Enter the **admin key**: the value of `edc.catalog.ui.asset.admin.key`.
3. What the panel offers depends on what the connector already holds:

| The connector has | The panel shows |
|---|---|
| No asset yet | The registration form (below). |
| One or more assets of the testbed type, none chosen (for example registered earlier with a script) | **Choose your testbed asset**: pick one with **Use this asset**. It is marked as the testbed asset, and its `no-constraint-policy` and contract definition are created if they are missing. Registering a second asset is refused. |
| A chosen testbed asset | The asset, with **Remove**. Remove it to register or choose a different one. |

The registration form takes:

| Field | Value |
|---|---|
| Asset id | The id the platform administrator gave you, if any. |
| S3 endpoint | The store, as seen from the connector, for example `http://rustfs:9000`. |
| Bucket | The bucket holding the datasets. |
| Access key and secret key | An access key of the testbed's own S3 store (created in its console). This is not the Data Lake key. |
| Prefix | Optional. Leave it empty to watch the whole bucket. |

**Register** creates, in one step, the asset, the `no-constraint-policy` and a contract definition that offers every asset under it.

The asset's data-address type is `6GDaliTestbedExperiments`. The panel works with assets of that type and says how many assets of other types it hides; **show them** lists them. An asset of another or an older type (for example `MinioFiles`) cannot be transferred by the connector's data plane: remove it and register the asset again.

Until the asset exists, the testbed offers nothing to the data space. Tell the platform administrator when it is registered.

---

## 5. Network access

This section applies to testbeds with their own connector.

A testbed connector must be reachable by the 6G-DALI Data Space connector over DSP. Publish it on the internet behind a TLS reverse proxy on a dedicated hostname, as the Data Space connector does (`edc.dataspace.6gdali.eu`). The connector keeps its ports. The proxy exposes them on 443, routed by path.

**Prerequisites**

- A public DNS name for the testbed connector, e.g. `edc.<testbed>.6gdali.eu`, pointing at the host.
- Inbound 443 open to the host.
- A TLS certificate for that name.

**Reverse proxy.** Use the nginx vhost from the connector bundle (`nginx/<domain>.conf`) and add the TLS certificate. It routes by path prefix and passes the path through untouched. The management API (18191) and the control API (18193) are deliberately not published.

| Path | Backend port | Expose publicly? |
|---|---|---|
| `/protocol*` | 18192 (DSP) | **Yes.** This is what the Data Space connector calls. |
| `/api*` | 18190 (observability, catalogue page) | Needed for the catalogue page and its Testbed asset panel (§4.3); changes there are protected by the admin key. Otherwise optional. |

**Connector configuration**

```properties
edc.dsp.callback.address=https://edc.<testbed>.6gdali.eu/protocol
```

Use `https://` throughout.

**Testbeds without a domain name.** A testbed with only a public IP address leaves out the nginx vhost and gives the administrator its IP address. Open 18192 (DSP) and 18190 (catalogue page) on the host, never 18191 or 18193, and set:

```properties
edc.dsp.callback.address=http://<testbed-ip>:18192/protocol
```

Either way, the DSP URL registered in the DataOps UI must be exactly the address the Data Space connector uses to reach the testbed's connector. A testbed that cannot be made reachable at all can be **hosted** instead (§6).

---

## 6. Hosted testbeds: the upload kit

A hosted testbed uploads with any S3 client to its upload bucket in the 6G-DALI Data Lake. The administrator hands over the **upload kit** (shown at the end of the registration wizard, and on the testbed page's Overview):

| Item | Example |
|---|---|
| Endpoint | the Data Lake's S3 endpoint |
| Bucket | `6g-dali-<testbed>-upload` |
| Access key | `tb-<testbed>-upload-…` |
| Secret key | shown once when the key is created; not stored by the platform |

The key can only list, read and write that one bucket. With the AWS CLI:

```bash
export AWS_ACCESS_KEY_ID=<access key>
export AWS_SECRET_ACCESS_KEY=<secret key>
aws s3 cp ./my-dataset/ s3://6g-dali-<testbed>-upload/my-dataset/ --recursive --endpoint-url <endpoint>
```

Any S3 client works (`mc`, `rclone`, boto3); use path-style addressing if the client asks.

- Lay out each dataset as in §7, `metadata.json` first.
- If the secret is lost, the testbed's administrators create a **new upload key** on the testbed page (Overview → *New upload key*). The old key stops working at once; the running transfer is not affected.
- Only files uploaded while the facility connector's transfer is running are picked up (§3.2). If an upload does not appear in the catalogue, ask the administrator to check the transfer.
- There are no transfer-completion messages for hosted testbeds (§10): check the catalogue instead.

---

## 7. Preparing and uploading a dataset

### 7.1 Layout in the bucket

```
<bucket>/
└── <dataset-dir>/            e.g. isi-ran-energy-2026-09
    ├── metadata.json         upload this first
    ├── part-1.csv
    └── part-2.csv
```

`<dataset-dir>` becomes part of the experiment ID. When it is not the identifier you want in the catalogue, set `identifier` in `metadata.json`.

### 7.2 Data files

The data files must satisfy the [Input Dataset Format](6G-DALI-input-dataset-format.md). The main rules are:

- UTF-8 with no BOM, comma-delimited, `.` as the decimal separator.
- A header line, with names in ASCII lowercase `snake_case`: they become the file's column list in the catalogue, and DataOps builds its quality checks from them.
- One row per timestamp for time series. Pivot or split long-format data first.
- No archives and no compression.

### 7.3 Upload

Upload `metadata.json` and the data files to the testbed's bucket (or upload bucket), under the dataset directory (§7.1), using your usual S3 tooling. Upload `metadata.json` first.

---

## 8. Dataset metadata (`metadata.json`)

`metadata.json` is the source of the dataset's catalogue record. The central platform turns it into DCAT-AP 3.0 plus GAIA-X and `dali:` properties. It stays in the bucket as the testbed shared it: edits made later in DataOps (§9) change the catalogue, not this file.

Mandatory fields: `title`, `description`, `issued`, `license`, `gdpr_compliant`, `fair_compliant`, `contains_pii`, `sns_project_name`. In practice the platform falls back to defaults for several of these.

A starting `metadata.json` template is provided in the connector bundle; the DataOps UI also has a **Metadata builder** (testbed page → Overview). The full field reference, including the `testbed_context` block based on the SNS-JU Common Metadata Template (CMT), is in the [Testbed Metadata JSON Guide](Testbed-Metadata-JSON-guide.md).

Rules to remember:

- `issued` and `temporal_*` use `YYYY-MM-DD`.
- `theme` and `language` take EU authority **codes** (`TECH`, `ENG`), not URLs.
- `license` is a licence URL or a known code: `CC-BY-4.0`, `CC-BY-SA-4.0`, `CC0-1.0`, `MIT`, `Apache-2.0` and other SPDX ids are turned into their URL.
- `produced_by` is preferably your organisation's GAIA-X participant IRI (ask the platform administrator). A plain organisation name is accepted and recorded as the organisation's name.
- `access_rights` **decides who sees the dataset when it is first registered**: `PUBLIC` everyone, `RESTRICTED` signed-in users, `NON_PUBLIC` the catalogue's members only. Without it, or with a value not in that list, the dataset is Members only. Changing it in the file later has no effect: ask an administrator (§9).
- Unknown keys are ignored. Both `snake_case` and `camelCase` are accepted.
- `testbed_context` is optional. Omit unknown fields rather than leaving `TODO` values.
- The file must be named `metadata.json` (case-insensitive) and sit in the dataset's directory. Other `.json` files are not treated as dataset metadata.

---

## 9. Visibility and access

**Catalogues.** Each testbed's catalogue is **public** or **private**, chosen when the testbed is registered and fixed once the catalogue exists. A catalogue's name, description and logo are always listed on the public catalogue site; a private one carries a *Members only* badge.

**Datasets** have one of three visibilities:

| Visibility | Who sees the dataset | `access_rights` in `metadata.json` |
|---|---|---|
| **Public** | Everyone, signed in or not. | `PUBLIC` |
| **Signed-in users** | Anyone signed in to 6G-DALI. | `RESTRICTED` |
| **Members only** | The catalogue's members and the Data Space operators. Everyone else neither finds nor opens it. | `NON_PUBLIC`, missing, or any other value |

- **A new dataset gets the visibility its `metadata.json` asks for** in `access_rights`, as in the table. This holds in public and private catalogues alike: a `PUBLIC` dataset in a private catalogue is public, while the catalogue keeps its *Members only* badge. The bundle's `template/metadata.json` and the metadata builder start with `PUBLIC`; set `NON_PUBLIC` for a dataset that must stay within the catalogue's members, test datasets included.
- **Changing the visibility later** is done by the testbed's administrators (or the platform administrators) on the testbed page's **Datasets** tab: *Edit* opens the dataset's page, where its visibility can be set to any of the three, in public and private catalogues alike. The same page edits its title, description, keywords and licence.
- A dataset keeps its visibility when it is registered again (a re-uploaded `metadata.json`, a re-registration by an administrator), even if `access_rights` has changed in the file. Re-registering does bring back the `metadata.json` fields, undoing edits made in DataOps.

**Members.** The testbed page's **Access** tab manages the catalogue's members, by e-mail address, at three levels:

| Level | Can |
|---|---|
| Reader | see the catalogue's Members-only datasets |
| Contributor | also create and edit datasets |
| Publisher | also delete and publish datasets |

People must have signed in to a 6G-DALI application once before they can be added. Changes apply within about 30 seconds; on the catalogue site, members see the Members-only datasets after signing in.

The same tab lists the testbed's **administrators** (managed by the platform administrators): the people who manage the testbed in DataOps, including its datasets and members.

---

## 10. Transfer-completion notifications (testbed-side)

This section applies to testbeds with their own connector.

The RabbitMQ connection is **internal to the testbed**. It lets the testbed know when a transfer has completed, for example to trigger its own post-processing or cleanup. It does not feed the central 6G-DALI platform. The broker is run and owned by the testbed, and the testbed decides whether to use it.

For every `.csv` file transferred through a `PiveauData` transfer, the testbed connector publishes a message to the configured queue (default exchange, durable).

```json
{"request_id": "isi-ran-energy-2026-09/dataset.csv", "status": "SUCCESS"}
```

| Field | Values | Meaning |
|---|---|---|
| `request_id` | `<dataset dir>/<file>.csv` | The source object key |
| `status` | `SUCCESS` or `FAILED` | `SUCCESS` once the file is in the Data Lake. `FAILED` if the S3 upload failed and the HTTP fallback was unavailable or also failed. |

Notes:

- `metadata.json` does not produce a notification.
- `SUCCESS` means the file is in the Data Lake. It does not say anything about the catalogue, which the central platform updates separately.
- The broker host, credentials and queue name are the testbed's choice. The existing stacks use a queue called `ds_connector`.
- If `edc.rabbitmq.host` or `.queue` is unset, notifications are skipped with a warning and the transfer does not fail.

---

## 11. Onboarding checklists

### 11.1 Testbed with its own connector

- [ ] Deploy the stack from the bundle: `docker compose up -d` and check `/api/check/health` (§4).
- [ ] Make the connector reachable (§5): create the DNS record, install the nginx vhost with TLS, expose `/protocol` (and `/api` for the catalogue page), and make sure `edc.dsp.callback.address` is the `https://` URL.
- [ ] Create an access key in the testbed's own S3 store for the connector.
- [ ] Register the bucket as an asset in the Testbed asset panel of the connector's catalogue page (§4.3) and tell the administrator.
- [ ] Set `edc.rabbitmq.*` only if the testbed wants completion messages from its own broker.
- [ ] Upload a small test dataset (`metadata.json`, then a CSV) and check, with the administrator, that:
  - the dataset appears on the testbed page's Datasets tab, with the visibility its `access_rights` asks for,
  - the file lands in the central Data Lake,
  - a `SUCCESS` message arrives on the testbed's own queue (if RabbitMQ is configured).

### 11.2 Hosted testbed

- [ ] Store the upload kit's secret key safely; it is shown once (§6).
- [ ] Upload a small test dataset (`metadata.json`, then a CSV) with the kit (§6, §7).
- [ ] Check, with the administrator, that the dataset appears on the testbed page's Datasets tab, with the visibility its `access_rights` asks for.

### 11.3 Platform administrator, for any testbed

- [ ] Register the testbed with the right connector choice and catalogue visibility, and provision it; every step must be green (§3.1, §3.2).
- [ ] Add the testbed's administrators (Access tab) and the catalogue's members (Access tab → Catalogue access).
- [ ] Add the testbed's Data Lake bucket to the asset monitor's watched buckets (`MONITOR_BUCKETS`), as provisioning reminds.
- [ ] With its own connector: Find asset, Negotiate contract, Start transfer (Assets tab). Hosted: check the Assets tab shows the transfer active.
- [ ] After the test upload, check the test dataset's visibility and change it if needed (§9).
