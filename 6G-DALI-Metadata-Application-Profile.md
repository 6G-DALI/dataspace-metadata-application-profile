# 6G-DALI Data Space — Metadata Application Profile

**Version:** 1.0
**Date:** 2026-03-13
**Project:** 6G-DALI (SNS-JU)
**Data Space Platform:** piveau-hub

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Standards and Specifications](#2-standards-and-specifications)
3. [Architecture Overview](#3-architecture-overview)
4. [Common Prefixes and Namespaces](#4-common-prefixes-and-namespaces)
5. [Dataset Metadata](#5-dataset-metadata)
   - 5.1 [Core DCAT-AP Fields](#51-core-dcat-ap-fields)
   - 5.2 [GAIA-X Extensions](#52-gaia-x-extensions)
   - 5.3 [5G/6G Testbed Context (CMT Extensions)](#53-5g6g-testbed-context-cmt-extensions)
   - 5.4 [Data Provenance (PROV-O)](#54-data-provenance-prov-o)
   - 5.5 [Data Quality (W3C DQV and Great Expectations)](#55-data-quality-w3c-dqv-and-great-expectations)
   - 5.6 [Distribution Metadata](#56-distribution-metadata)
6. [Data Service Metadata (DataOps)](#6-data-service-metadata-dataops)
7. [ML Model Metadata (MLOps / MLDCAT-AP)](#7-ml-model-metadata-mlops--mldcat-ap)
8. [Data Catalog](#8-data-catalog)
9. [Field Reference Tables](#9-field-reference-tables)
   - 9.1 [CMT → DCAT-AP Field Mapping](#91-cmt--dcat-ap-field-mapping)
   - 9.2 [MRS Slices_V0_2 → DCAT-AP Field Mapping](#92-mrs-slices_v02--dcat-ap-field-mapping)
   - 9.3 [Obligation Levels Summary](#93-obligation-levels-summary)
10. [RDF Examples](#10-rdf-examples)
    - 10.1 [Dataset Example](#101-dataset-example)
    - 10.2 [Data Service Example](#102-data-service-example)
    - 10.3 [ML Model Example](#103-ml-model-example)
11. [piveau-hub Deployment Considerations](#11-piveau-hub-deployment-considerations)

---

## 1. Introduction

This document defines the metadata application profile for the **6G-DALI Data Space**, a federated data infrastructure for sharing and consuming datasets, data services, and ML models produced within the 6G-DALI project and associated 5G/6G testbeds.

Three categories of resources are described:

| Resource type | Lifecycle stage | Primary standard |
|---|---|---|
| **Dataset** | Produced by 5G/6G testbeds; ingested into the Data Lake | DCAT-AP 3.0 + GAIA-X + CMT |
| **Data Service** | DataOps transformations, quality checks, and augmentations applied to datasets | DCAT-AP 3.0 `dcat:DataService` |
| **ML Model** | Models trained in MLOps pipelines consuming datasets | MLDCAT-AP + DCAT-AP 3.0 |

All metadata records are stored in the **piveau-hub** instance (the Data Space catalogue) as RDF documents in Turtle format and must conform to:

- [DCAT-AP 3.0](https://semiceu.github.io/DCAT-AP/releases/3.0.0/) — the EU application profile of W3C DCAT
- [GAIA-X Trust Framework](https://gaia-x.eu/policy-rules-committee/trust-framework/) — for interoperability with European data spaces
- SNS-JU **Common Metadata Template (CMT) V1.0** — 5G/6G domain-specific fields
- **SLICES RI MRS** `Slices_V0_2` profile — for cross-project dataset exchange with the SLICES-RI metadata repository
- [W3C Data Quality Vocabulary (DQV)](https://www.w3.org/TR/vocab-dqv/) — for quality annotations
- [MLDCAT-AP](https://semiceu.github.io/MLDCAT-AP/releases/2.0.0/) — for ML model descriptions
- [W3C PROV-O](https://www.w3.org/TR/prov-o/) — for provenance lineage

---

## 2. Standards and Specifications

| Standard | Scope | Reference |
|---|---|---|
| W3C DCAT 3.0 | Dataset, DataService, Catalog vocabulary | https://www.w3.org/TR/vocab-dcat-3/ |
| DCAT-AP 3.0 | EU application profile of DCAT | https://semiceu.github.io/DCAT-AP/releases/3.0.0/ |
| MLDCAT-AP | Application profile for ML models and datasets | https://semiceu.github.io/MLDCAT-AP/releases/2.0.0/ |
| GAIA-X Trust Framework | European data space interoperability | https://gaia-x.eu/policy-rules-committee/trust-framework/ |
| GAIA-X Ontology | Self-descriptions and compliance | `gax-trust-framework:` namespace |
| W3C DQV | Data quality metrics and annotations | https://www.w3.org/TR/vocab-dqv/ |
| W3C PROV-O | Provenance and lineage | https://www.w3.org/TR/prov-o/ |
| SNS-JU CMT V1.0 | 5G/6G dataset identity, content, and metrics | SNS-JU internal document |
| SLICES RI MRS Slices_V0_2 | Metadata profile for SLICES RI interoperability | SLICES-RI DMI |
| FAIR Principles | Findable, Accessible, Interoperable, Reusable | https://www.go-fair.org/fair-principles/ |
| Dublin Core Terms (DCT) | General metadata properties | https://www.dublincore.org/specifications/dublin-core/dcmi-terms/ |
| ADMS | Asset Description Metadata Schema | https://www.w3.org/TR/vocab-adms/ |
| Schema.org | Scientific dataset extensions | https://schema.org/ |
| Great Expectations | Data quality expectation suites | https://greatexpectations.io/ |
| 3GPP TS 28.552 | 5G NRM measurement families | 3GPP |
| ETSI TR 103 761 | Dataset characterization for AI/ML in networks | ETSI |

---

## 3. Architecture Overview

```
                          data + metadata
  ┌────────────┐  ──────────────────────────────────────────────┐
  │  5G/6G     │        raw data                                 ▼
  │  Testbeds  │ ──────────────────────────────▶ ┌─────────────────────┐
  └────────────┘                                 │     Data Lake       │
        │                                        │     (Storage)       │
        │ metadata (dcat:Dataset)                └──────────┬──────────┘
        ▼                                                   │ raw data
  ┌──────────────────────────────────────────┐             │
  │          piveau-hub                      │        ┌────▼────────┐
  │       (Catalogue / Portal)               │        │   DataOps   │
  │                                          │        │  Pipelines  │
  │  dcat:Catalog                            │        └────┬────────┘
  │  ├─ dcat:Dataset        ◀── metadata ───────────────┐ │
  │  │   (raw + derived)      (derived datasets,        │ │ processed data
  │  │                         quality annotations)     │ │
  │  ├─ dcat:DataService    ◀── metadata ───────────────┘ │
  │  │   (DataOps services)                               │
  │  │                                              ┌─────▼───────┐
  │  └─ mldcat:MLModel      ◀── metadata ──────────│   MLOps     │
  │      (trained models)                           │  Pipelines  │
  │                                                 └─────────────┘
  └──────────────────────────────────────────┘

Resource lifecycle:
  1. Testbed generates raw dataset → data uploaded to Data Lake
                                  → metadata (dcat:Dataset) registered in piveau-hub
  2. DataOps reads data from Data Lake → runs transformations and quality checks
     → registers dcat:DataService in piveau-hub
     → registers derived dcat:Dataset (with prov:wasDerivedFrom) in piveau-hub
     → adds dqv:QualityMeasurement annotations to datasets in piveau-hub
  3. MLOps reads processed data from Data Lake → trains models
     → registers mldcat:MLModel (with performance metrics) in piveau-hub
```

All three resource types share the same **piveau-hub** catalogue instance and are discoverable via SPARQL and the REST API. The catalogue is GAIA-X compliant and federated with the broader European data space ecosystem.

---

## 4. Common Prefixes and Namespaces

The following RDF prefixes are used throughout this specification:

```turtle
PREFIX adms:     <http://www.w3.org/ns/adms#>
PREFIX dcat:     <http://www.w3.org/ns/dcat#>
PREFIX dct:      <http://purl.org/dc/terms/>
PREFIX dqv:      <http://www.w3.org/ns/dqv#>
PREFIX foaf:     <http://xmlns.com/foaf/0.1/>
PREFIX gax:      <https://registry.lab.gaia-x.eu/v1/api/trusted-shape-registry/v1/shapes/jsonld/trustframework#>
PREFIX mldcat:   <http://www.w3.org/ns/mldcat#>
PREFIX owl:      <http://www.w3.org/2002/07/owl#>
PREFIX prov:     <http://www.w3.org/ns/prov#>
PREFIX rdf:      <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs:     <http://www.w3.org/2000/01/rdf-schema#>
PREFIX schema:   <https://schema.org/>
PREFIX skos:     <http://www.w3.org/2004/02/skos/core#>
PREFIX vcard:    <http://www.w3.org/2006/vcard/ns#>
PREFIX xsd:      <http://www.w3.org/2001/XMLSchema#>

# 6G-DALI project-specific extension namespace
PREFIX dali:     <https://dali-project.eu/ns#>
```

The base IRI for resources in the 6G-DALI Data Space follows:

```
https://dataspace.6gdali.eu/set/data/{resource-uuid}        # datasets
https://dataspace.6gdali.eu/set/service/{resource-uuid}     # data services
https://dataspace.6gdali.eu/set/model/{resource-uuid}       # ML models
https://dataspace.6gdali.eu/set/distribution/{uuid}         # distributions
https://dataspace.6gdali.eu/catalogue                       # catalog root
```

---

## 5. Dataset Metadata

A **dataset** in the 6G-DALI Data Space is described as a `dcat:Dataset` resource. Datasets are generated by 5G and 6G testbeds, uploaded to the Data Lake, and their metadata registered in piveau-hub. They may also be produced as outputs of DataOps pipelines (derived datasets).

### 5.1 Core DCAT-AP Fields

The following table lists all DCAT-AP properties for `dcat:Dataset`. Obligation levels are **M** (Mandatory), **R** (Recommended), and **O** (Optional).

| Property | Predicate | Obligation | Description |
|---|---|---|---|
| Title | `dct:title` | M | Human-readable name of the dataset. Use language tags (`@en`). |
| Description | `dct:description` | M | Free-text description of the dataset content. |
| Identifier | `dct:identifier` | M | Unique identifier string (e.g., DOI, UUID, accession number). |
| Issued | `dct:issued` | M | Date the dataset was first published (`xsd:date`). |
| Modified | `dct:modified` | R | Date of last modification (`xsd:date`). |
| Publisher | `dct:publisher` | R | Organisation responsible for making the dataset available (`foaf:Organization`). |
| Creator | `dct:creator` | R | Entity that created the dataset (`foaf:Person` or `foaf:Organization`). |
| Contact point | `dcat:contactPoint` | R | Contact information (`vcard:Organization` or `vcard:Individual`). |
| Theme / Category | `dcat:theme` | R | Category from a controlled vocabulary (EU Data Theme, SKOS). |
| Keyword | `dcat:keyword` | R | Free keyword tags (`xsd:string` with language tag). |
| Access rights | `dct:accessRights` | M | Access condition using EU Access Right vocabulary. |
| License | `dct:license` | M | License URI (e.g., CC-BY-4.0). |
| Language | `dct:language` | R | Language of the dataset content (EU MDR Languages vocabulary). |
| Spatial coverage | `dct:spatial` | R | Geographic coverage (URI or `dct:Location`). |
| Temporal coverage | `dct:temporal` | R | Time period covered (`dct:PeriodOfTime` with `dcat:startDate` / `dcat:endDate`). |
| Landing page | `dcat:landingPage` | R | Web page for human access to the dataset. |
| Distribution | `dcat:distribution` | R | Link to one or more `dcat:Distribution` resources. |
| Version | `adms:version` | R | Version label of the dataset. |
| Is part of series | `dct:isPartOf` | O | Reference to a `dcat:DatasetSeries`. |
| Is version of | `dct:isVersionOf` | O | Reference to the original resource this is a version of. |
| Conforms to | `dct:conformsTo` | R | Standard or specification the dataset conforms to (e.g., FAIR Principles URI). |
| Provenance | `dct:provenance` | R | Provenance statement (`dct:ProvenanceStatement`). |
| Relation | `dct:relation` | O | Related resource (publications, code repositories, etc.). |
| Rights | `dct:rights` | O | Rights statement beyond the license. |
| Bibliographic citation | `dct:bibliographicCitation` | O | Citation text for the dataset. |
| Byte size | `dcat:byteSize` | R | Approximate total size (note: use on `dcat:Distribution` for exact size). |
| Qualified attribution | `prov:qualifiedAttribution` | R | Attribution to a `prov:Agent` with a role. |
| Was derived from | `prov:wasDerivedFrom` | O | Source dataset(s) this dataset was derived from. |
| ADMS identifier | `adms:identifier` | R | Structured identifier (e.g., DOI via `adms:Identifier`). |

> **FAIR compliance note:** All datasets must declare conformance to the FAIR Principles using `dct:conformsTo <https://www.go-fair.org/fair-principles/>`. Persistent identifiers (DOI preferred) must be provided via `adms:identifier`.

### 5.2 GAIA-X Extensions

For GAIA-X compliance, datasets must carry a **self-description** that includes the GAIA-X trust framework properties. These are expressed using the `gax:` prefix alongside the DCAT-AP description.

| Property | Predicate | Obligation | Description |
|---|---|---|---|
| GAIA-X type | `rdf:type gax:DataResource` | M | Declares the resource as a GAIA-X Data Resource. |
| Exposed through | `gax:exposedThrough` | M | Reference to the `gax:DataExchangeComponent` (the Data Space endpoint) through which the dataset is accessible. |
| Contains PII | `gax:containsPII` | M | Boolean indicating whether the dataset contains Personally Identifiable Information (`xsd:boolean`). |
| Legal basis | `gax:legalBasis` | R | Legal basis for processing (e.g., GDPR Article 6 reference). |
| Produced by | `gax:producedBy` | M | Reference to the `gax:LegalParticipant` that produced the dataset. |
| Policy | `gax:policy` | R | ODRL or usage policy expression governing access. |
| Checksum | `spdx:checksum` | R | Cryptographic checksum of the data for integrity verification. |

> **GDPR/FAIR compliance declaration:** Every dataset must include a boolean assertion (`dali:gdprCompliant`, `dali:fairCompliant`) confirming the owner's compliance statement, as required by the SNS-JU CMT.

```turtle
<dataset-uri>
    rdf:type  dcat:Dataset, gax:DataResource ;
    dali:gdprCompliant    true^^xsd:boolean ;
    dali:fairCompliant    true^^xsd:boolean ;
    gax:containsPII       false^^xsd:boolean ;
    gax:producedBy        <https://dali-project.eu/participant/ku-leuven> .
```

### 5.3 5G/6G Testbed Context (CMT Extensions)

The SNS-JU Common Metadata Template (CMT) defines 5G/6G-specific fields that are not covered by standard DCAT-AP. These are mapped using the `dali:` project namespace and `schema:` where applicable.

#### A. Dataset Identity (CMT Group 1)

| CMT Field | Predicate | Value type | Example |
|---|---|---|---|
| SNS Project Name | `dali:snsProjectName` | `xsd:string` | `"6G-DALI"` |
| Owner Name | `dct:publisher` → `foaf:name` | `foaf:Organization` | `"IMEC"` |
| Owner Contact email | `dcat:contactPoint` → `vcard:hasEmail` | `vcard:Email` | `mailto:contact@imec.be` |
| List of Contributors | `dct:contributor` | `foaf:Agent` | — |
| Related Publications | `dct:relation` / `schema:citation` | `xsd:anyURI` | DOI or arXiv URI |
| Domain Keywords | `dcat:keyword` | `xsd:string` | `"6G"@en`, `"RAN"@en` |

#### B. Dataset Object Characteristics (CMT Group 2)

| CMT Field | Predicate | Value type | Example |
|---|---|---|---|
| GDPR & FAIR compliance | `dali:gdprCompliant`, `dali:fairCompliant` | `xsd:boolean` | `true` |
| License type | `dct:license` | URI | `<https://creativecommons.org/licenses/by/4.0/>` |
| External dataset link | `dcat:distribution` → `dcat:accessURL` | URI | Zenodo, S3, MinIO URL |

#### C. Dataset Content — Underlay Network & Compute (CMT Group 3A)

These fields capture the testbed infrastructure context and are represented using the `dali:` namespace under a `dali:TestbedContext` blank node or named resource.

```turtle
<dataset-uri>
    dali:testbedContext [
        rdf:type                    dali:TestbedContext ;

        # Infrastructure
        dali:underlayPlatform       <https://example.testbed.eu/platform> ;
        dali:environment            "urban" ;           # indoors | urban | rural | mixed
        dali:networkDomain          "RAN" ;             # RAN | Transport | CORE | E2E

        # RAN parameters
        dali:ran3gppRelease         "Release 17" ;
        dali:ranNewRadioType        "NR-SA" ;           # NR-SA | NR-NSA | LTE
        dali:ranSplit               "DU-RU split" ;     # DU-RU split | No-Split | CU-DU split
        dali:ranFocusedTechnology   "O-RAN" ;           # O-RAN | JSAC | RIC | No_focus
        dali:ranCoverageType        "Single_Macro" ;    # Single_Macro | Single_Micro | Multicell_setup
        dali:ranFrequencyBand       "n78" ;
        dali:ranBandwidthMHz        100 ;
        dali:ranMaxEndDevices       10 ;
        dali:ranMobilityModel       "pedestrian" ;      # static | pedestrian | vehicular | UAV

        # Core parameters
        dali:coreRelease            "Release 17" ;
        dali:coreSolution           "OpenSource" ;      # OpenSource | Commercial

        # Transport parameters
        dali:transportType          "fiber_optics" ;    # wired | microwave | fiber_optics | satellite

        # Compute parameters
        dali:computeOrchestratorType   "Kubernetes" ;  # Kubernetes | OpenStack | OSM | ONAP
        dali:computeGpuUse             false ;
        dali:computeVirtualizationType "Docker" ;       # KVM | Docker | Bare-metal
        dali:computeInfrastructureType "private-edge node" ;
    ] .
```

#### D. Dataset Content — Service Description (CMT Group 3B)

| CMT Field | Predicate | Values |
|---|---|---|
| Traffic origin | `dali:trafficOrigin` | `Manual` \| `Application` |
| Traffic pattern | `dali:trafficPattern` | `UL/DL UDP` \| `DL TCP` \| etc. |
| Slice type | `dali:sliceType` | `slice101` \| `Multi-slice` \| `No_slicing` |
| Reference plane | `dali:referencePlane` | `control plane` \| `data plane` \| `management plane` |
| Related vertical | `dali:relatedVertical` | `Vertical_agnostic` \| `CAM` \| `HEALTH` \| etc. |

#### E. Dataset Content — Experimentation Process & Metrics (CMT Group 3C)

| CMT Field | Predicate | Values / Notes |
|---|---|---|
| Observation point (horizontal) | `dali:observationPointHorizontal` | `E2E Application layer` \| `End device to Access` \| `DU to CU` \| `Access to Edge` \| `Access to Core` \| `Core to Cloud` (per CMT Annex 1) |
| Observation point (vertical) | `dali:observationPointVertical` | `Radio Level` \| `Network Layer` \| `Application Layer` \| `Compute Resource-level` \| `cross-layer` |
| Measurement family | `dali:measurementFamily` | Values from 3GPP TS 28.552: `DRB` \| `RRC` \| `RRU` \| `L1M` \| `PEE` \| etc. (see CMT Annex 2) |
| Measurement tools | `dali:measurementTool` | `"tcpdump"` \| `"Prometheus exporter"` \| etc. |
| Measured metrics | `schema:variableMeasured` | Free text or structured; use one per metric: `"Throughput (Kbps)"` |
| Measurement technique | `schema:measurementTechnique` | Description of the measurement method |

### 5.4 Data Provenance (PROV-O)

Provenance chains must be maintained throughout the dataset lifecycle. Use W3C PROV-O to link datasets to their origins.

```turtle
<derived-dataset-uri>
    prov:wasDerivedFrom     <source-dataset-uri> ;
    prov:wasGeneratedBy     <activity-uri> ;
    prov:wasAttributedTo    <agent-uri> .

<activity-uri>
    rdf:type            prov:Activity ;
    prov:startedAtTime  "2025-06-01T08:00:00Z"^^xsd:dateTime ;
    prov:endedAtTime    "2025-06-01T09:30:00Z"^^xsd:dateTime ;
    prov:used           <source-dataset-uri> ;
    prov:wasAssociatedWith <service-uri> ;
    rdfs:label          "DataOps ETL Pipeline Run"@en .
```

For datasets uploaded directly from testbeds:

```turtle
<dataset-uri>
    dct:provenance [
        rdf:type            dct:ProvenanceStatement ;
        rdfs:label          "Collected by the KU Leuven MaMIMO testbed during experiment campaign 2025-Q1. Uploaded via AF REST API."@en
    ] ;
    prov:wasAttributedTo [
        rdf:type      prov:Agent, foaf:Organization ;
        foaf:name     "KU Leuven WaveCoRE" ;
        foaf:homepage <https://www.esat.kuleuven.be/wavecore/>
    ] .
```

### 5.5 Data Quality (W3C DQV and Great Expectations)

Data quality is described using the **W3C Data Quality Vocabulary (DQV)** and can be populated from **Great Expectations** validation results.

#### 5.5.1 DQV Core Concepts

| DQV Concept | Description |
|---|---|
| `dqv:QualityMeasurement` | A concrete measurement of a metric on a dataset |
| `dqv:Metric` | The specific quality criterion being measured |
| `dqv:Dimension` | A high-level quality aspect (Completeness, Accuracy, Timeliness, etc.) |
| `dqv:QualityAnnotation` | A human or automated annotation of quality |
| `dqv:QualityPolicy` | A set of rules/expectations the dataset should satisfy |

#### 5.5.2 Recommended Quality Dimensions

| Dimension | `dqv:Dimension` URI | Description |
|---|---|---|
| Completeness | `dali:dqv/completeness` | Proportion of non-null/non-missing values |
| Accuracy | `dali:dqv/accuracy` | Correctness of values against reference |
| Timeliness | `dali:dqv/timeliness` | Freshness of data relative to collection time |
| Consistency | `dali:dqv/consistency` | Adherence to schema and value constraints |
| Uniqueness | `dali:dqv/uniqueness` | Absence of duplicate records |
| Validity | `dali:dqv/validity` | Conformance to defined formats and ranges |

#### 5.5.3 Great Expectations Mapping to DQV

The following mapping translates Great Expectations concepts to DQV:

| Great Expectations concept | DQV equivalent | Notes |
|---|---|---|
| Expectation Suite | `dqv:QualityPolicy` | Named set of quality rules |
| Expectation | `dqv:Metric` | Individual quality rule (e.g., `expect_column_values_to_not_be_null`) |
| Validation Result | `dqv:QualityMeasurement` | Result of running expectations on data |
| `observed_value` | `dqv:value` | Numeric value of the measurement |
| `success: true/false` | `dqv:isMeasurementOf` + result annotation | Pass/fail outcome |
| Data Docs HTML | `dqv:QualityAnnotation` body | Human-readable quality report |

#### 5.5.4 RDF Pattern for Quality Annotations

```turtle
# Quality Policy (Great Expectations Suite)
<https://dataspace.6gdali.eu/quality/suite/dataset-abc>
    rdf:type                dqv:QualityPolicy ;
    rdfs:label              "6G Dataset Quality Expectations"@en ;
    dct:description         "Great Expectations suite for 5G/6G measurement datasets"@en ;
    dct:creator             <https://dali-project.eu/participant/sparkworks> .

# A quality measurement instance
<https://dataspace.6gdali.eu/quality/measurement/dataset-abc/001>
    rdf:type                dqv:QualityMeasurement ;
    dqv:isMeasurementOf     dali:metric/completeness-csi-column ;
    dqv:computedOn          <dataset-uri> ;
    dqv:value               "0.998"^^xsd:decimal ;
    dct:date                "2025-06-01"^^xsd:date ;
    prov:wasGeneratedBy     <https://dataspace.6gdali.eu/service/quality-checker> .

# The metric definition
dali:metric/completeness-csi-column
    rdf:type                dqv:Metric ;
    rdfs:label              "CSI column completeness"@en ;
    dct:description         "Fraction of non-null values in the CSI measurement column (GE: expect_column_values_to_not_be_null)"@en ;
    dqv:inDimension         dali:dqv/completeness ;
    dqv:expectedDataType    xsd:decimal .

# Attach quality annotations to the dataset
<dataset-uri>
    dqv:hasQualityMeasurement  <https://dataspace.6gdali.eu/quality/measurement/dataset-abc/001> ;
    dqv:hasQualityAnnotation   [
        rdf:type            dqv:QualityAnnotation ;
        oa:hasBody          <https://dataspace.6gdali.eu/quality/report/dataset-abc.html> ;
        oa:motivatedBy      dqv:qualityAssessment ;
        dct:date            "2025-06-01"^^xsd:date ;
        rdfs:comment        "Validation passed: 99.8% completeness. 2 rows with null CSI values flagged."@en
    ] .
```

> **Note:** The `oa:` prefix is `http://www.w3.org/ns/oa#` (W3C Web Annotation Vocabulary), used by DQV for quality annotations.

### 5.6 Distribution Metadata

Each `dcat:Dataset` must have at least one `dcat:Distribution`. A distribution describes a specific downloadable or accessible form of the dataset.

| Property | Predicate | Obligation | Description |
|---|---|---|---|
| Title | `dct:title` | M | Human-readable name of the distribution. |
| Description | `dct:description` | R | Description of what this distribution contains. |
| Access URL | `dcat:accessURL` | M | URL to access the resource (landing page or API endpoint). |
| Download URL | `dcat:downloadURL` | R | Direct download URL (if available). |
| Format | `dct:format` | R | File format (EU MDR File Type vocabulary URI). |
| Media type | `dcat:mediaType` | R | IANA media type string (e.g., `application/json`). |
| License | `dct:license` | R | License applicable to this distribution. |
| Byte size | `dcat:byteSize` | R | File size in bytes (`xsd:nonNegativeInteger`). |
| Checksum | `spdx:checksum` | R | Cryptographic checksum for integrity. |
| Compression format | `dcat:compressFormat` | O | Compression applied (e.g., `application/gzip`). |
| Encoding format | `dcat:packageFormat` | O | Encoding format (e.g., `UTF-8`). |
| Conformance | `dct:conformsTo` | O | Data standard or schema the distribution conforms to. |
| Availability | `dcatap:availability` | R | Expected availability (`dcatap:STABLE`, `dcatap:AVAILABLE`, etc.). |

---

## 6. Data Service Metadata (DataOps)

**Data services** represent DataOps processes — transformations, quality checks, augmentation pipelines, or API endpoints — that operate on datasets. They are modelled as `dcat:DataService` resources in DCAT-AP.

A data service may:
- **Consume** input datasets (`dcat:servesDataset`)
- **Produce** derived datasets (linked via `prov:wasDerivedFrom` on the output dataset)
- **Expose** a processing API endpoint (`dcat:endpointURL`)

### 6.1 Data Service Fields

| Property | Predicate | Obligation | Description |
|---|---|---|---|
| Title | `dct:title` | M | Service name. |
| Description | `dct:description` | M | What the service does to the data. |
| Endpoint URL | `dcat:endpointURL` | M | URL of the service endpoint (API or processing trigger). |
| Endpoint description | `dcat:endpointDescription` | R | OpenAPI or other specification describing the endpoint. |
| Serves dataset | `dcat:servesDataset` | R | Input dataset(s) that this service operates on. |
| License | `dct:license` | M | License for use of this service. |
| Publisher | `dct:publisher` | M | Organisation operating this service. |
| Contact point | `dcat:contactPoint` | R | Contact information. |
| Theme | `dcat:theme` | R | Category. |
| Access rights | `dct:accessRights` | M | Who can invoke the service. |
| Conforms to | `dct:conformsTo` | R | Specification the service conforms to (e.g., OpenAPI 3.0 URI). |
| Service type | `dali:serviceType` | M | Nature of the service: `Transformation` \| `QualityCheck` \| `Augmentation` \| `Aggregation` \| `Anonymisation` \| `Feature Engineering` |
| Input format | `dali:inputFormat` | R | Expected input data format. |
| Output format | `dali:outputFormat` | R | Produced output data format. |
| Framework | `dali:framework` | O | Processing framework used (e.g., `"Apache Spark"`, `"Apache Kafka"`, `"Great Expectations"`). |
| Version | `adms:version` | R | Version of the service software. |
| Source code | `schema:codeRepository` | O | Link to service source code repository. |

### 6.2 Data Service RDF Pattern

```turtle
<https://dataspace.6gdali.eu/set/service/{uuid}>
    rdf:type                dcat:DataService ;
    dct:title               "CSI Feature Extraction Service"@en ;
    dct:description         "Transforms raw CSI binary files into tabular feature vectors for ML training. Applies PCA dimensionality reduction and normalization."@en ;
    dcat:endpointURL        <https://dataops.dali-project.eu/api/v1/csi-features> ;
    dcat:endpointDescription <https://dataops.dali-project.eu/api/v1/openapi.json> ;
    dcat:servesDataset      <https://dataspace.6gdali.eu/set/data/{input-dataset-uuid}> ;
    dct:license             <https://opensource.org/licenses/MIT> ;
    dct:publisher           [ rdf:type foaf:Organization ; foaf:name "SparkWorks P.C." ] ;
    dct:accessRights        <http://publications.europa.eu/resource/authority/access-right/RESTRICTED> ;
    dali:serviceType        "Feature Engineering" ;
    dali:inputFormat        "application/octet-stream" ;
    dali:outputFormat       "text/csv" ;
    dali:framework          "Apache Spark" ;
    adms:version            "1.2.0" ;
    schema:codeRepository   <https://github.com/dali-project/csi-feature-extractor> .
```

---

## 7. ML Model Metadata (MLOps / MLDCAT-AP)

ML models produced by 6G-DALI MLOps pipelines are described in the Data Space using the **MLDCAT-AP** application profile, which extends DCAT-AP with properties specific to machine learning artefacts.

An ML model is typed as both `dcat:Dataset` (for catalogue compatibility with piveau-hub) and `mldcat:MLModel`.

### 7.1 Core Model Fields

| Property | Predicate | Obligation | Description |
|---|---|---|---|
| Title | `dct:title` | M | Human-readable model name. |
| Description | `dct:description` | M | Model purpose and architecture description. |
| Identifier | `dct:identifier` | M | Unique identifier (UUID or DOI). |
| Version | `adms:version` | M | Semantic version of the model (e.g., `1.0.0`). |
| Issued | `dct:issued` | M | Model publication date. |
| Publisher | `dct:publisher` | M | Organisation that published the model. |
| Creator | `dct:creator` | R | Person(s) who created/trained the model. |
| License | `dct:license` | M | Model license. |
| Access rights | `dct:accessRights` | M | Access conditions. |
| Landing page | `dcat:landingPage` | R | Human-readable model card page. |
| Distribution | `dcat:distribution` | M | Distribution(s) of the model artefact (weights files). |

### 7.2 MLDCAT-AP Specific Fields

| Property | Predicate | Obligation | Description |
|---|---|---|---|
| ML model type | `mldcat:mlModelType` | M | Algorithm family: `"Classification"` \| `"Regression"` \| `"Clustering"` \| `"Reinforcement Learning"` \| `"Federated Learning"` \| `"Generative"` etc. |
| ML task | `mldcat:mlTask` | M | Specific task: `"Localization"` \| `"Beamforming Prediction"` \| `"Anomaly Detection"` \| `"Channel Estimation"` \| `"QoS Prediction"` etc. |
| Training dataset | `mldcat:trainedOn` | M | Reference to the `dcat:Dataset` used for training. |
| Evaluation dataset | `mldcat:evaluatedOn` | R | Reference to the `dcat:Dataset` used for evaluation/testing. |
| ML framework | `mldcat:mlFramework` | R | Software framework: `"TensorFlow"` \| `"PyTorch"` \| `"scikit-learn"` \| `"Keras"` etc. |
| ML framework version | `mldcat:mlFrameworkVersion` | R | Framework version string. |
| Input features | `mldcat:inputFeatures` | R | Description or schema of the model input features. |
| Output features | `mldcat:outputFeatures` | R | Description or schema of the model outputs/predictions. |
| Hyperparameters | `mldcat:hyperparameters` | R | Key training hyperparameters (JSON literal or named resource). |
| Was trained using | `prov:wasGeneratedBy` | R | Reference to the MLOps training `prov:Activity`. |
| Training dataset split | `dali:trainingSplit` | R | Fraction used for training (e.g., `0.8`). |
| Hardware used | `dali:trainingHardware` | O | Hardware platform for training (e.g., `"NVIDIA A100 GPU"`). |
| Compute environment | `dali:trainingComputeEnv` | O | Training infrastructure (e.g., `"Kubernetes cluster"`, `"HPC cluster"`). |

### 7.3 Model Performance Metrics

Model performance metrics are expressed using **W3C DQV** (applied to the model evaluation context) to maintain consistency with dataset quality annotations.

| Metric | `dqv:Metric` URI | Description |
|---|---|---|
| Accuracy | `dali:metric/model-accuracy` | Classification accuracy on test set |
| F1 Score | `dali:metric/model-f1` | F1 score (macro or per-class) |
| Precision | `dali:metric/model-precision` | Precision on test set |
| Recall | `dali:metric/model-recall` | Recall on test set |
| MAE | `dali:metric/model-mae` | Mean Absolute Error (regression tasks) |
| RMSE | `dali:metric/model-rmse` | Root Mean Square Error (regression tasks) |
| Localization Error | `dali:metric/localization-error-m` | Mean localization error in metres (6G-specific) |
| Inference Latency | `dali:metric/inference-latency-ms` | Model inference time in milliseconds |
| Model Size | `dali:metric/model-size-mb` | Model artefact size in MB |

```turtle
# Performance measurement on the model
<https://dataspace.6gdali.eu/quality/measurement/model-xyz/f1>
    rdf:type            dqv:QualityMeasurement ;
    dqv:isMeasurementOf dali:metric/model-f1 ;
    dqv:computedOn      <model-uri> ;
    dqv:value           "0.94"^^xsd:decimal ;
    dct:description     "Macro F1 score on held-out test set (20% split)"@en ;
    dct:date            "2025-07-15"^^xsd:date .
```

### 7.4 Model Fairness and Bias

For models used in sensitive applications, add bias/fairness annotations:

| Property | Predicate | Description |
|---|---|---|
| Bias mitigation | `dali:biasMitigationApplied` | Boolean or description of techniques used |
| Protected attributes | `dali:protectedAttributes` | List of demographic attributes checked |
| Fairness metric | `dqv:QualityMeasurement` with `dali:metric/model-fairness-*` | Quantitative fairness measure |

### 7.5 Model Distribution

Model artefacts (weights, ONNX files, containerised models) are described as `dcat:Distribution` resources with model-specific media types:

| Format | `dcat:mediaType` | Description |
|---|---|---|
| ONNX | `application/onnx` | Open Neural Network Exchange format |
| TensorFlow SavedModel | `application/zip` | TF SavedModel directory (zipped) |
| PyTorch | `application/octet-stream` | `.pt` or `.pth` file |
| scikit-learn | `application/octet-stream` | Pickle or joblib file |
| PMML | `application/xml` | Predictive Model Markup Language |
| Container image | `application/vnd.oci.image.manifest.v1+json` | Docker/OCI container image |

---

## 8. Data Catalog

The piveau-hub instance exposes a **top-level catalogue** that aggregates all datasets, data services, and ML models registered in the 6G-DALI Data Space.

```turtle
<https://dataspace.6gdali.eu/catalogue>
    rdf:type            dcat:Catalog ;
    dct:title           "6G-DALI Data Space Catalogue"@en ;
    dct:description     "Federated catalogue of datasets, data services, and ML models from the 6G-DALI project, generated by 5G and 6G testbeds across Europe."@en ;
    dct:issued          "2025-01-01"^^xsd:date ;
    dct:modified        "2026-03-01"^^xsd:date ;
    dct:publisher       [ rdf:type  foaf:Organization ;
                          foaf:name "6G-DALI Consortium" ;
                          foaf:homepage <https://dali-project.eu> ] ;
    dct:language        <http://publications.europa.eu/resource/authority/language/ENG> ;
    dct:license         <https://creativecommons.org/licenses/by/4.0/> ;
    dcat:themeTaxonomy  <http://publications.europa.eu/resource/authority/data-theme> ;
    dcat:dataset        <dataset-1-uri>, <dataset-2-uri> ;   # links to all datasets
    dcat:service        <service-1-uri>, <service-2-uri> ;   # links to data services
    dcat:record         <record-1-uri> .                     # catalogue records (provenance of registration)
```

Sub-catalogues may be created per testbed or partner organisation using `dcat:Catalog` with `dct:isPartOf` linking to the root catalogue.

---

## 9. Field Reference Tables

### 9.1 CMT → DCAT-AP Field Mapping

| CMT Field | CMT Group | DCAT-AP / Extension Predicate |
|---|---|---|
| Identifier | Identity | `dct:identifier` |
| Version | Identity | `adms:version` |
| Name of Dataset | Identity | `dct:title` |
| Date | Identity | `dct:issued` |
| SNS Project Name | Identity | `dali:snsProjectName` |
| Owner Name | Identity | `dct:publisher` → `foaf:name` |
| Owner Contact email | Identity | `dcat:contactPoint` → `vcard:hasEmail` |
| List of Contributors | Identity | `dct:contributor` |
| Dataset Short Description | Identity | `dct:description` |
| Related Publications | Identity | `dct:relation`, `schema:citation` |
| Keywords | Identity | `dcat:keyword` |
| GDPR & FAIR compliance | Object | `dali:gdprCompliant`, `dali:fairCompliant` |
| License type | Object | `dct:license` |
| External dataset link | Object | `dcat:distribution` → `dcat:accessURL` |
| Underlay Platform | Content-A | `dali:testbedContext` → `dali:underlayPlatform` |
| Environment | Content-A | `dali:testbedContext` → `dali:environment` |
| Network domain | Content-A | `dali:testbedContext` → `dali:networkDomain` |
| RAN 3GPP Release | Content-A | `dali:testbedContext` → `dali:ran3gppRelease` |
| RAN NR Type | Content-A | `dali:testbedContext` → `dali:ranNewRadioType` |
| RAN split | Content-A | `dali:testbedContext` → `dali:ranSplit` |
| RAN technology | Content-A | `dali:testbedContext` → `dali:ranFocusedTechnology` |
| RAN coverage | Content-A | `dali:testbedContext` → `dali:ranCoverageType` |
| RAN frequency band | Content-A | `dali:testbedContext` → `dali:ranFrequencyBand` |
| RAN bandwidth | Content-A | `dali:testbedContext` → `dali:ranBandwidthMHz` |
| RAN max end devices | Content-A | `dali:testbedContext` → `dali:ranMaxEndDevices` |
| RAN mobility model | Content-A | `dali:testbedContext` → `dali:ranMobilityModel` |
| CORE release | Content-A | `dali:testbedContext` → `dali:coreRelease` |
| CORE solution | Content-A | `dali:testbedContext` → `dali:coreSolution` |
| Transport type | Content-A | `dali:testbedContext` → `dali:transportType` |
| Compute orchestrator | Content-A | `dali:testbedContext` → `dali:computeOrchestratorType` |
| Compute GPU use | Content-A | `dali:testbedContext` → `dali:computeGpuUse` |
| Compute virtualisation | Content-A | `dali:testbedContext` → `dali:computeVirtualizationType` |
| Traffic origin | Content-B | `dali:trafficOrigin` |
| Traffic pattern | Content-B | `dali:trafficPattern` |
| Slice type | Content-B | `dali:sliceType` |
| Reference plane | Content-B | `dali:referencePlane` |
| Related vertical | Content-B | `dali:relatedVertical` |
| Observation point (H) | Content-C | `dali:observationPointHorizontal` |
| Observation point (V) | Content-C | `dali:observationPointVertical` |
| Measurement family | Content-C | `dali:measurementFamily` |
| Measurement tools | Content-C | `dali:measurementTool` |
| Measured indicators | Content-C | `schema:variableMeasured` |

### 9.2 MRS Slices_V0_2 → DCAT-AP Field Mapping

For interoperability with the SLICES RI Metadata Repository Service (MRS), the following mapping translates the `Slices_V0_2` JSON profile to DCAT-AP RDF:

| MRS Field | DCAT-AP Predicate | Notes |
|---|---|---|
| `identifier` | `dct:identifier` | UUID |
| `name` | `dct:title` | |
| `description` | `dct:description` | |
| `scientificDomains` | `dcat:theme` | Map to EU Data Theme vocabulary |
| `scientificSubdomains` | `dcat:keyword` | |
| `dateTimeStart` | `dcat:startDate` (in `dct:temporal`) | |
| `dateTimeEnd` | `dcat:endDate` (in `dct:temporal`) | |
| `createdAt` | `dct:issued` | |
| `version` | `adms:version` | |
| `byteSize` | `dcat:byteSize` | |
| `format` | `dct:format` | |
| `compressionFormat` | `dcat:compressFormat` | |
| `encodingFormat` | `dcat:packageFormat` | |
| `creators` | `dct:creator` (→ `foaf:Person`) | Map firstName+lastName → `foaf:name`, email → `foaf:mbox`, organization → `schema:affiliation` |
| `locations` | `dct:spatial` / `schema:geo` | Map lat/long → `schema:GeoCoordinates`, country → EU Countries vocabulary |
| `accessType` | `dct:accessRights` | Map to EU Access Right vocabulary |
| `accessMode` | `dcat:accessService` or custom property | |
| `license` | `dct:license` | Map to SPDX or CC license URI |
| `licenseUri` | `dct:license` (as URI) | |
| `provenance` | `dct:provenance` / `prov:wasDerivedFrom` | |
| `keywords` | `dcat:keyword` | |
| `rightsUri` | `dct:rights` | |

### 9.3 Obligation Levels Summary

The following matrix summarises which fields are required for each resource type:

| Field | Dataset | Data Service | ML Model |
|---|---|---|---|
| `dct:title` | **M** | **M** | **M** |
| `dct:description` | **M** | **M** | **M** |
| `dct:identifier` | **M** | **M** | **M** |
| `dct:issued` | **M** | R | **M** |
| `dct:license` | **M** | **M** | **M** |
| `dct:accessRights` | **M** | **M** | **M** |
| `dct:publisher` | R | **M** | **M** |
| `dct:creator` | R | — | R |
| `dcat:contactPoint` | R | R | R |
| `dcat:theme` | R | R | R |
| `dcat:keyword` | R | — | R |
| `dcat:distribution` | R | — | **M** |
| `dcat:endpointURL` | — | **M** | — |
| `dcat:servesDataset` | — | R | — |
| `adms:version` | R | R | **M** |
| `dct:conformsTo` | R | R | — |
| `dali:gdprCompliant` | **M** | — | — |
| `dali:fairCompliant` | **M** | — | — |
| `dali:snsProjectName` | **M** | — | — |
| `dali:testbedContext` | R | — | — |
| `dqv:hasQualityMeasurement` | R | — | R |
| `prov:wasDerivedFrom` | R* | — | — |
| `mldcat:mlModelType` | — | — | **M** |
| `mldcat:mlTask` | — | — | **M** |
| `mldcat:trainedOn` | — | — | **M** |
| `mldcat:mlFramework` | — | — | R |
| `mldcat:hyperparameters` | — | — | R |

*R only for derived datasets produced by DataOps pipelines.

---

## 10. RDF Examples

### 10.1 Dataset Example

A 5G/6G testbed dataset with full DCAT-AP, GAIA-X, CMT, provenance, and quality annotations:

```turtle
PREFIX adms:   <http://www.w3.org/ns/adms#>
PREFIX dcat:   <http://www.w3.org/ns/dcat#>
PREFIX dct:    <http://purl.org/dc/terms/>
PREFIX dqv:    <http://www.w3.org/ns/dqv#>
PREFIX foaf:   <http://xmlns.com/foaf/0.1/>
PREFIX gax:    <https://registry.lab.gaia-x.eu/v1/api/trusted-shape-registry/v1/shapes/jsonld/trustframework#>
PREFIX prov:   <http://www.w3.org/ns/prov#>
PREFIX rdf:    <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX schema: <https://schema.org/>
PREFIX skos:   <http://www.w3.org/2004/02/skos/core#>
PREFIX vcard:  <http://www.w3.org/2006/vcard/ns#>
PREFIX xsd:    <http://www.w3.org/2001/XMLSchema#>
PREFIX dali:   <https://dali-project.eu/ns#>

<https://dataspace.6gdali.eu/set/data/a1b2c3d4-0000-0000-0000-000000000001>
    rdf:type                     dcat:Dataset, gax:DataResource ;

    # --- Core Identity ---
    dct:title                    "5G RAN QoS Measurements — Urban Scenario Q1 2025"@en ;
    dct:description              "Comprehensive 5G NR QoS measurement dataset collected in an urban macro-cell environment using a Release 17 standalone deployment. Contains throughput, latency, and RSRP measurements across 50 UE positions with pedestrian and vehicular mobility patterns."@en ;
    dct:identifier               "a1b2c3d4-0000-0000-0000-000000000001" ;
    dct:issued                   "2025-04-01"^^xsd:date ;
    dct:modified                 "2025-04-15"^^xsd:date ;
    dct:language                 <http://publications.europa.eu/resource/authority/language/ENG> ;
    adms:version                 "1.0" ;
    adms:identifier              [ rdf:type            adms:Identifier ;
                                   skos:notation       "10.48804/A1B2C3D4" ;
                                   adms:schemeAgency   "DataCite" ] ;

    # --- DCAT-AP Mandatory ---
    dct:accessRights             <http://publications.europa.eu/resource/authority/access-right/PUBLIC> ;
    dct:license                  <https://creativecommons.org/licenses/by/4.0/> ;

    # --- Publisher & Contacts ---
    dct:publisher                [ rdf:type       foaf:Organization ;
                                   foaf:name      "ExampleTelco Testbed Partner" ;
                                   foaf:homepage  <https://testbed.example.eu> ] ;
    dct:creator                  [ rdf:type            foaf:Person ;
                                   foaf:name           "Jane Doe" ;
                                   foaf:mbox           <mailto:jane.doe@testbed.example.eu> ;
                                   schema:affiliation  "ExampleTelco" ] ;
    dcat:contactPoint            [ rdf:type        vcard:Organization ;
                                   vcard:fn        "ExampleTelco 5G Lab" ;
                                   vcard:hasEmail  <mailto:5glab@testbed.example.eu> ;
                                   vcard:hasURL    <https://testbed.example.eu/lab> ] ;

    # --- Categorisation ---
    dcat:theme                   <http://publications.europa.eu/resource/authority/data-theme/TECH> ;
    dcat:keyword                 "5G"@en, "NR-SA"@en, "QoS"@en, "RAN"@en,
                                 "throughput"@en, "latency"@en, "urban"@en, "pedestrian"@en ;
    dcat:landingPage             <https://dataspace.6gdali.eu/dataset/a1b2c3d4> ;

    # --- Spatial & Temporal ---
    dct:spatial                  <http://publications.europa.eu/resource/authority/country/GRC> ;
    dct:temporal                 [ rdf:type        dct:PeriodOfTime ;
                                   dcat:startDate  "2025-01-10"^^xsd:date ;
                                   dcat:endDate    "2025-03-31"^^xsd:date ] ;
    schema:geo                   [ rdf:type          schema:GeoCoordinates ;
                                   schema:address    "Athens, Greece" ;
                                   schema:latitude   "37.9838" ;
                                   schema:longitude  "23.7275" ] ;

    # --- Standards Conformance ---
    dct:conformsTo               <https://www.go-fair.org/fair-principles/> ;
    dct:bibliographicCitation    "Doe, J. et al. (2025). 5G RAN QoS Measurements Urban Scenario." ;
    dct:relation                 <https://arxiv.org/abs/2025.00000> ;

    # --- SNS-JU / DALI specific ---
    dali:snsProjectName          "6G-DALI" ;
    dali:gdprCompliant           true^^xsd:boolean ;
    dali:fairCompliant           true^^xsd:boolean ;

    # --- GAIA-X ---
    gax:containsPII              false^^xsd:boolean ;
    gax:producedBy               <https://dali-project.eu/participant/example-telco> ;

    # --- Testbed Context (CMT) ---
    dali:testbedContext          [
        rdf:type                    dali:TestbedContext ;
        dali:underlayPlatform       <https://testbed.example.eu/platform/athens-5g> ;
        dali:environment            "urban" ;
        dali:networkDomain          "RAN" ;
        dali:ran3gppRelease         "Release 17" ;
        dali:ranNewRadioType        "NR-SA" ;
        dali:ranSplit               "DU-RU split" ;
        dali:ranFocusedTechnology   "O-RAN" ;
        dali:ranCoverageType        "Single_Macro" ;
        dali:ranFrequencyBand       "n78" ;
        dali:ranBandwidthMHz        100 ;
        dali:ranMaxEndDevices       50 ;
        dali:ranMobilityModel       "pedestrian" ;
        dali:computeOrchestratorType "Kubernetes" ;
        dali:computeGpuUse          false^^xsd:boolean ;
        dali:computeVirtualizationType "Docker" ;
        dali:trafficOrigin          "Application" ;
        dali:trafficPattern         "DL UDP" ;
        dali:sliceType              "slice101" ;
        dali:referencePlane         "data plane" ;
        dali:relatedVertical        "Vertical_agnostic" ;
        dali:observationPointHorizontal "Access node to network core" ;
        dali:observationPointVertical   "Network Layer" ;
        dali:measurementFamily      "DRB", "RRU", "QF" ;
        dali:measurementTool        "Prometheus exporter", "tcpdump" ;
    ] ;

    # --- Measured Variables ---
    schema:variableMeasured      "Throughput (Mbps)", "Latency (ms)", "RSRP (dBm)", "SINR (dB)" ;
    schema:measurementTechnique  "Prometheus-based 5G NR KPI monitoring with 1s scrape interval"@en ;

    # --- Provenance ---
    dct:provenance               [ rdf:type   dct:ProvenanceStatement ;
                                   rdfs:label "Collected via automated AF-MRS export. Experiment ID: 351. Raw data from eBOS database."@en ] ;
    prov:wasAttributedTo         <https://dali-project.eu/participant/example-telco> ;

    # --- Funding ---
    schema:funding               [ rdf:type           schema:Grant ;
                                   schema:funder      "EU Horizon Europe" ;
                                   schema:identifier  "101000000" ;
                                   schema:name        "6G-DALI Project" ] ;

    # --- Quality ---
    dqv:hasQualityMeasurement    <https://dataspace.6gdali.eu/quality/measurement/a1b2c3d4/completeness> ;

    # --- Distributions ---
    dcat:byteSize                "524288000"^^xsd:nonNegativeInteger ;
    dcat:distribution            <https://dataspace.6gdali.eu/set/distribution/dist-001>,
                                 <https://dataspace.6gdali.eu/set/distribution/dist-002> .

# Distribution 1 — CSV data file
<https://dataspace.6gdali.eu/set/distribution/dist-001>
    rdf:type            dcat:Distribution ;
    dct:title           "QoS Measurement CSV"@en ;
    dct:description     "Tabular CSV file with per-UE QoS measurements"@en ;
    dct:format          <http://publications.europa.eu/resource/authority/file-type/CSV> ;
    dct:license         <https://creativecommons.org/licenses/by/4.0/> ;
    dcat:accessURL      <https://datalake.dali-project.eu/datasets/a1b2c3d4/data.csv> ;
    dcat:downloadURL    <https://datalake.dali-project.eu/datasets/a1b2c3d4/data.csv> ;
    dcat:mediaType      "text/csv" ;
    dcat:byteSize       "524288000"^^xsd:nonNegativeInteger .

# Distribution 2 — Data documentation
<https://dataspace.6gdali.eu/set/distribution/dist-002>
    rdf:type            dcat:Distribution ;
    dct:title           "Dataset README"@en ;
    dct:description     "Column descriptions, units, and collection methodology"@en ;
    dct:format          <http://publications.europa.eu/resource/authority/file-type/TXT> ;
    dct:license         <https://creativecommons.org/licenses/by/4.0/> ;
    dcat:accessURL      <https://datalake.dali-project.eu/datasets/a1b2c3d4/README.md> ;
    dcat:mediaType      "text/markdown" .
```

### 10.2 Data Service Example

```turtle
<https://dataspace.6gdali.eu/set/service/s9s8s7s6-0000-0000-0000-000000000001>
    rdf:type                dcat:DataService ;
    dct:title               "5G QoS Data Quality Checker"@en ;
    dct:description         "DataOps quality gate service that runs a Great Expectations validation suite on 5G QoS datasets. Checks for missing values, out-of-range KPIs, and schema conformance. Annotates the input dataset with DQV quality measurements."@en ;
    dcat:endpointURL        <https://dataops.dali-project.eu/api/v1/quality-check> ;
    dcat:endpointDescription <https://dataops.dali-project.eu/api/v1/openapi.json> ;
    dcat:servesDataset      <https://dataspace.6gdali.eu/set/data/a1b2c3d4-0000-0000-0000-000000000001> ;
    dct:license             <https://opensource.org/licenses/Apache-2.0> ;
    dct:accessRights        <http://publications.europa.eu/resource/authority/access-right/RESTRICTED> ;
    dct:publisher           [ rdf:type foaf:Organization ; foaf:name "SparkWorks P.C." ] ;
    dcat:theme              <http://publications.europa.eu/resource/authority/data-theme/TECH> ;
    dali:serviceType        "QualityCheck" ;
    dali:inputFormat        "text/csv" ;
    dali:outputFormat       "application/json" ;
    dali:framework          "Great Expectations 1.x" ;
    adms:version            "2.1.0" ;
    schema:codeRepository   <https://github.com/dali-project/qos-quality-checker> .
```

### 10.3 ML Model Example

```turtle
PREFIX mldcat: <http://www.w3.org/ns/mldcat#>

<https://dataspace.6gdali.eu/set/model/m1m2m3m4-0000-0000-0000-000000000001>
    rdf:type                dcat:Dataset, mldcat:MLModel ;

    dct:title               "5G UE Localization CNN Model"@en ;
    dct:description         "Convolutional Neural Network for UE indoor localization using CSI fingerprinting. Trained on the KU Leuven MaMIMO CSI dataset. Achieves 18-21mm mean localization error."@en ;
    dct:identifier          "m1m2m3m4-0000-0000-0000-000000000001" ;
    dct:issued              "2025-07-01"^^xsd:date ;
    adms:version            "1.0.0" ;
    dct:license             <https://opensource.org/licenses/MIT> ;
    dct:accessRights        <http://publications.europa.eu/resource/authority/access-right/PUBLIC> ;

    dct:publisher           [ rdf:type foaf:Organization ; foaf:name "KU Leuven WaveCoRE" ] ;
    dct:creator             [ rdf:type foaf:Person ;
                              foaf:name "Jane Researcher" ;
                              foaf:mbox <mailto:jane@kuleuven.be> ] ;
    dcat:contactPoint       [ rdf:type vcard:Organization ;
                              vcard:fn "KU Leuven WaveCoRE" ;
                              vcard:hasEmail <mailto:wavecore@kuleuven.be> ] ;

    dcat:keyword            "localization"@en, "CSI"@en, "CNN"@en,
                            "5G"@en, "Massive MIMO"@en, "indoor positioning"@en ;
    dcat:theme              <http://publications.europa.eu/resource/authority/data-theme/TECH> ;
    dcat:landingPage        <https://dataspace.6gdali.eu/model/m1m2m3m4> ;

    # --- MLDCAT-AP specific ---
    mldcat:mlModelType      "Classification" ;
    mldcat:mlTask           "Localization" ;
    mldcat:mlFramework      "TensorFlow" ;
    mldcat:mlFrameworkVersion "2.13.0" ;
    mldcat:trainedOn        <https://dataspace.6gdali.eu/set/data/kul-mimo-csi-dataset> ;
    mldcat:evaluatedOn      <https://dataspace.6gdali.eu/set/data/kul-mimo-csi-dataset-test> ;
    mldcat:inputFeatures    "CSI matrix: 64×100 complex float32 values per sample"@en ;
    mldcat:outputFeatures   "2D position (x, y) in metres within measurement grid"@en ;
    mldcat:hyperparameters  """{"layers": 5, "filters": [32,64,128,64,32], "dropout": 0.3,
                               "optimizer": "Adam", "lr": 0.001, "epochs": 100,
                               "batch_size": 128}"""^^xsd:string ;

    # --- Provenance ---
    prov:wasGeneratedBy     [ rdf:type prov:Activity ;
                              rdfs:label "MLOps training pipeline run 2025-06-30"@en ;
                              prov:startedAtTime "2025-06-30T08:00:00Z"^^xsd:dateTime ;
                              prov:endedAtTime   "2025-06-30T14:22:00Z"^^xsd:dateTime ] ;
    prov:wasDerivedFrom     <https://dataspace.6gdali.eu/set/data/kul-mimo-csi-dataset> ;

    # --- Training infrastructure ---
    dali:trainingHardware       "NVIDIA A100 40GB GPU (x4)" ;
    dali:trainingComputeEnv     "Kubernetes cluster" ;
    dali:trainingSplit          "0.8"^^xsd:decimal ;

    # --- Performance Metrics (DQV) ---
    dqv:hasQualityMeasurement
        <https://dataspace.6gdali.eu/quality/measurement/m1m2m3m4/loc-error>,
        <https://dataspace.6gdali.eu/quality/measurement/m1m2m3m4/model-size> ;

    # --- Distribution (model artefact) ---
    dcat:distribution       <https://dataspace.6gdali.eu/set/distribution/model-dist-001> .

<https://dataspace.6gdali.eu/set/distribution/model-dist-001>
    rdf:type            dcat:Distribution ;
    dct:title           "CSI Localization CNN — ONNX weights"@en ;
    dct:description     "Serialised ONNX model file for inference deployment"@en ;
    dct:license         <https://opensource.org/licenses/MIT> ;
    dcat:accessURL      <https://datalake.dali-project.eu/models/m1m2m3m4/model.onnx> ;
    dcat:downloadURL    <https://datalake.dali-project.eu/models/m1m2m3m4/model.onnx> ;
    dcat:mediaType      "application/onnx" ;
    dcat:byteSize       "48234567"^^xsd:nonNegativeInteger .

# Quality measurements
<https://dataspace.6gdali.eu/quality/measurement/m1m2m3m4/loc-error>
    rdf:type            dqv:QualityMeasurement ;
    dqv:isMeasurementOf dali:metric/localization-error-m ;
    dqv:computedOn      <https://dataspace.6gdali.eu/set/model/m1m2m3m4-0000-0000-0000-000000000001> ;
    dqv:value           "0.019"^^xsd:decimal ;
    dct:description     "Mean localization error in metres on test set (21,000 samples)"@en ;
    dct:date            "2025-07-01"^^xsd:date .
```

---

## 11. piveau-hub Deployment Considerations

### 11.1 Catalogue API

piveau-hub exposes a DCAT-AP compliant REST API. Datasets, data services, and ML models are registered using HTTP PUT/POST requests with Turtle or JSON-LD payloads:

```
PUT  /datasets/{catalogue-id}/{dataset-id}    Content-Type: text/turtle
PUT  /catalogues/{catalogue-id}               Content-Type: text/turtle
GET  /datasets/{catalogue-id}/{dataset-id}    Accept: text/turtle
```

### 11.2 Required DCAT-AP Extensions

piveau-hub expects the following DCAT-AP EU extensions in registrations:

- `dcatap:availability` on `dcat:Distribution` — use values from the DCAT-AP availability vocabulary (`http://data.europa.eu/r5r/availability/`)
- EU MDR controlled vocabularies for `dct:format`, `dct:language`, `dct:spatial`, `dct:accessRights`, `dcat:theme`

### 11.3 Catalogue Identifier

Each resource must belong to a catalogue declared in piveau-hub. The 6G-DALI catalogue identifier is:

```
Catalogue ID: 6g-dali
Base URL:     https://dataspace.6gdali.eu/catalogue/6g-dali
```

Sub-catalogues per partner testbed should use IDs following the pattern `6g-dali-{partner-short-name}` (e.g., `6g-dali-kuleuven`, `6g-dali-sparkworks`).

### 11.4 GAIA-X Federated Catalogue Integration

For federation with the GAIA-X Federated Catalogue, each resource self-description must be signed using a valid GAIA-X participant credential. The signing process uses the GAIA-X Compliance Service and results in a `gx:compliance` claim attached to the self-description JSON-LD document.

### 11.5 MRS / SLICES-RI Cross-Registration

Datasets that are also to be registered in the SLICES RI MRS (`Slices_V0_2` profile) should use the AF-MRS integration pipeline:

1. Export the DCAT-AP RDF record from piveau-hub
2. Transform to `Slices_V0_2` JSON using the field mapping in [Section 9.2](#92-mrs-slices_v02--dcat-ap-field-mapping)
3. Authenticate with Keycloak (`client_credentials` flow) and POST to the MRS `/datasets` endpoint
4. Upload the dataset file to the returned pre-signed URL via HTTP PUT with explicit `Content-Length`

The `dct:identifier` (UUID) must be consistent between the piveau-hub registration and the MRS `identifier` field to enable cross-system linking.

### 11.6 Custom Entity Support via SHACL

piveau-hub natively handles `dcat:Dataset` and has partial support for `dcat:DataService`. Resources typed as `mldcat:MLModel` are accepted as datasets but piveau-hub has no built-in understanding of their additional properties.

To extend piveau-hub with proper validation and (where supported) UI handling for these entity types, custom SHACL shapes must be defined as a project deliverable:

| Entity | Status in piveau-hub | Required SHACL work |
|---|---|---|
| `dcat:Dataset` | Native support | Extend with `dali:` property constraints |
| `dcat:DataService` | Partial (DCAT-AP 3.0) | Define mandatory `dali:serviceType`, input/output format shapes |
| `mldcat:MLModel` | Registered as dataset | Define full node shape for `mldcat:` properties and performance metrics |
| `dali:TestbedContext` | None (custom class) | Define shape for all CMT 5G/6G context properties |
| `dqv:QualityMeasurement` | None (custom class) | Define shape linking measurements to datasets and metrics |

The SHACL shapes should be layered as follows:

```
dcat-ap-3.0.shapes.ttl          ← upstream SEMIC shapes (unmodified)
    └── dali-base.shapes.ttl    ← dali: property constraints on dcat:Dataset
         ├── dali-service.shapes.ttl    ← DataService extensions
         ├── dali-model.shapes.ttl      ← MLModel / MLDCAT-AP extensions
         └── dali-quality.shapes.ttl    ← DQV quality measurement shapes
```

These shapes serve a dual purpose: validation of incoming metadata payloads before registration, and documentation of the mandatory/recommended fields per entity type.

### 11.7 Validation

All metadata records should be validated against:

- **DCAT-AP 3.0 SHACL shapes** — available from the SEMIC SHACL validator
- **6G-DALI SHACL profile** — the layered shapes described in Section 11.6 (to be defined as a project deliverable)
- **MLDCAT-AP SHACL shapes** — for ML model records

---

*End of document*