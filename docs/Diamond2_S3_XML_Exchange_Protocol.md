# TV Programme Feed — S3 Exchange Protocol

## Change History

| Version | Date | Change |
| --- | --- | --- |
| 1.1 | 27/11/2025 | Final Word (.docx) revision. |
| 1.2 | 29/7/2026 | Converted from Word to Markdown to simplify change tracking and enable external contributions via GitHub.<br>Corrected the XML envelope example to match the published schemas (`Document` root element, `urn:cdn:pdx:v1` namespace).<br>Fixed section numbering and cross-references.<br>Expanded the filename rules with guidance on naming patterns and processing order. |
| 1.3 | 29/7/2026 | Corrected the bucket layout: live and staging are separate dedicated buckets (not subdirectories of a shared sender root), each with incoming/, complete/ and errors/ at the top level. |
| 1.4 | 24/9/2026 | Defined the completion status file and error report (§6.1, §6.2). Both are XML documents in the `urn:tep:pdx:report:1.0` namespace, defined by the TEP report schema `tep-pdx-report-v1.xsd`, now published alongside the Diamond 2 data schemas. Replaced the "to be published separately" placeholders with the report structure, envelope attributes, per-record rules, error severity and the error code contract. |

## 1. Purpose

Define a deterministic, fault-tolerant process for exchanging programme level metadata between **Senders** (broadcasters or their agents) and the **Receiver** (CDN Programme Data Exchange platform) using Amazon S3 object storage.

## 2. Authentication

Access to the S3 bucket may be granted via one of two methods. The preferred approach is for TEP to grant permission to an existing IAM role maintained by the broadcaster within their own AWS environment; this enables the broadcaster to manage their credentials in accordance with their existing security policies. Alternatively, TEP can generate a dedicated access key pair with appropriate permissions to the bucket and provide these credentials securely to the broadcaster. Each broadcaster will be consulted regarding their preferred authentication method during the onboarding phase.

See the [S3 Authentication guide](Diamond2_S3_Authentication.md) for full details of the recommended cross-account IAM role approach.

## 3. System Expectations

The following limits are envisaged more as per-broadcaster default rate limits:

- Maximum XML files processed per day: 1,000 files
- Maximum episodes ingested per day: 10,000 episodes
- Maximum episodes per single XML file: 1,000 episodes
- Maximum single XML file size: 100 MB
- Maximum ingest rate: 100 files every 15 minutes

## 4. Actors & Responsibilities

| Actor | Responsibilities |
| --- | --- |
| **Sender** | • Generate XML files containing new/changed episodes.<br>• Ensure filenames are unique for this broadcaster **and** lexically incremental (see §5.2).<br>• Never alter or delete objects after upload. |
| **Receiver** | • Own and administer the S3 bucket (encryption, IAM, lifecycle).<br>• Poll for new files every 15 minutes (or use S3 events).<br>• Validate each file and ingest it, then either emit a status file into `complete/`, **or** move the XML plus an error report into `errors/`. |

## 5. Bucket Layout

Each broadcaster has two dedicated S3 buckets: one for live data exchange and one for staging/testing. The bucket names are provided during onboarding (see the [S3 Authentication guide](Diamond2_S3_Authentication.md)). Both buckets have the same layout, with the three exchange directories at the top level:

```
<bucket>/
│
├── incoming/              # freshly uploaded objects
│   └── file_000126.xml
│
├── complete/              # successfully ingested
│   └── file_000124.status.xml
│
└── errors/                # rejected files + diagnostics
    ├── file_000123.xml
    └── file_000123.errors.xml
```

### 5.1. Retention

- `incoming/` — files will remain here until they have been processed by the Receiver.
- `complete/` — the Receiver will remove files from here 7 days after each file is created.
- `errors/` — the Receiver will remove files from here 7 days after each file is created.

### 5.2. Filename rules

The Sender may choose any naming scheme (e.g. zero-padded integer, timestamp) **as long as**:

1. The name is unique within the bucket.
2. Later files sort *after* earlier ones using plain lexical ordering.

#### Processing order and naming patterns

Files are processed in lexical order of filename. Each time the system looks for work it takes whichever file sorts first among those waiting, so the name you give a file determines the order in which it is ingested, and a name that sorts badly will cause files to be processed in an order you did not intend.

Four patterns to avoid:

- **Prefixing with `PRE_TX` and `POST_TX`.** POST sorts before PRE, so your Post-TX file will be processed first. A publication record referencing an episode ID that TEP has not yet seen is silently ignored (see the [Field Reference](Diamond2_XML_Field_Reference_and_Implementation_Notes.md)), so this will quietly discard transmission records with no error raised anywhere (the only trace is a missing entry in the completion status file, see §6.2). If you are prefixing this way, it is worth a look at what has actually landed.
- **A GUID at the start of the filename.** GUIDs are effectively random, so files are processed in an arbitrary order. If you need a GUID for your own reconciliation, or to ensure file name uniqueness, put it at the end of the name rather than the beginning.
- **Day-first dates.** A date written 14-07-2026 sorts by day before month, so 01-12-2026 will be processed before 02-01-2026. Use YYYY-MM-DD or YYYYMMDD, which sort correctly as text.
- **Unpadded sequence numbers.** file2 sorts after file10. Pad to a fixed width, so file0002 and file0010.

Examples that will not work as intended:

```
POST_TX_2026-07-14.xml
PRE_TX_2026-07-14.xml
3f9a2c14-8b7e-4d02-9c1f-5a6e7b8d9012_episodes.xml
pretx_14-07-2026.xml
pretx_2.xml, pretx_10.xml
```

Examples that will work better:

```
2026-07-14-0930-01-pretx.xml
2026-07-14-0930-02-posttx.xml
20260714-093000-pretx-3f9a2c14.xml
20260714-093001-posttx-3f9a2c14.xml
pretx-0002.xml, pretx-0010.xml
```

The general principle is to lead with a date or timestamp in a format that sorts correctly, then anything else you need for your own purposes, and to make sure that where a Pre-TX and a Post-TX file are dependent on each other the Pre-TX file sorts first.

## 6. Polling & Processing Sequence

1. **Discovery** — every 15 minutes the Receiver lists `incoming/*.xml` in lexical order.
2. **Validation** — the Receiver does XML schema based validation and any other application-specific validation on each incoming XML file.
3. **Processing** — on success, the Receiver moves the valid files out of `incoming/` and processes them in **strict lexical order**.
4. **Completion** — exactly one report is written for each processed file. In the names below `<base>` is the uploaded filename with its `.xml` extension removed, so `2026-07-14-0930-01-pretx.xml` produces `2026-07-14-0930-01-pretx.status.xml` or `2026-07-14-0930-01-pretx.errors.xml`.
   - On success ➜ write `complete/<base>.status.xml` (see §6.2). The uploaded file is deleted from `incoming/`; it is **not** copied to `complete/`, which holds only status files.
   - On failure ➜ move the uploaded XML unchanged to `errors/<base>.xml` and write `errors/<base>.errors.xml` beside it (see §6.1).

Both reports are XML documents in the namespace `urn:tep:pdx:report:1.0`. This is deliberately separate from the `urn:cdn:pdx:v1` namespace of the files a Sender uploads, so that the reports and the data schemas can be revised independently of one another. The normative definition of both reports is the report schema [tep-pdx-report-v1.xsd](tep-pdx-report-v1.xsd), which is authored and maintained by TEP and published alongside the Diamond 2 data schemas. The schema is annotated throughout and is the reference for anything not covered in this section. A report can be validated against it in the usual way:

```bash
xmllint --schema tep-pdx-report-v1.xsd --noout 2026-07-14-0930-01-pretx.status.xml
```

The root element of each report carries the same three attributes:

| Attribute | Meaning |
| --- | --- |
| `schemaVersion` | Revision of the report schema that produced the report. Currently `1.0`. |
| `file` | Bare name of the uploaded file the report is about (e.g. `2026-07-14-0930-01-pretx.xml`), so that a report can be matched to the file that produced it. |
| `processedAt` | UTC date-time at which processing of the file finished. This is not the time the file was uploaded. |

Two things never appear in either report. Where a submitted value was dropped because another Commissioner holds a lock on that field (see §6.2), the report says which field was affected but never who holds the lock, since that party is frequently a competitor of the Sender. And the reports carry no internal diagnostic context from TEP's processing pipeline: the attributes on each `<error>` and `<warning>` element are the fixed set defined by the schema.

### 6.1. Error Handling

**All-or-nothing:** if *any* record is invalid, the **entire file** is rejected and nothing in it is written to TEP. The error report therefore always describes a file that changed nothing, however many of its records were individually fine, and its job is to identify the records that need to be fixed.

The root element is `<errorReport>`. In addition to the three common attributes it carries `stage`, the stage of processing at which the file failed (for example `VALIDATING` or `PERSISTING`), and `summary`, a one-line human-readable statement of why the file was rejected. Both are intended for a person reading the report. `stage` in particular is free text that TEP may extend, so do not branch on it; branch on the error codes instead.

The report has two optional child elements, each of which is omitted when it would be empty:

- `<file>` holds faults that belong to the document as a whole rather than to any one record: the file could not be parsed, failed XSD validation, or declared an unsupported namespace or schema version.
- `<programmes>` holds faults attributable to individual records, as one `<programme>` element per record that caused the rejection. Its `id` attribute is the Sender's own Project `id` as it appeared in the file. The attribute is optional here because a record can fail before its identifier has been read, in which case the fault is still reported as a `<programme>` element but without an `id`. Records that were themselves fine are not listed, so that the records needing attention are not buried among them.

Each fault is an `<error>` element with the following attributes:

| Attribute | Meaning |
| --- | --- |
| `code` | Stable machine-readable identifier for the kind of error (required). See *Error codes* below. |
| `message` | Human-readable explanation in English, intended for a person triaging the file (required). Do not parse it. |
| `severity` | `error` — the submitted data was rejected and needs to be corrected before the file is resubmitted.<br>`critical` — processing itself failed on the TEP side. The data may be valid, and resubmitting the file unchanged is reasonable. (required) |
| `field` | Name of the field the error concerns, where the fault is attributable to a single field (optional). |
| `line` | 1-based line number in the uploaded file, where the fault can be placed (optional). |

Warnings (see §6.2) never appear on an error report: nothing in a rejected file was written, so there is no applied change for a warning to describe.

Example:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<errorReport xmlns="urn:tep:pdx:report:1.0" schemaVersion="1.0" file="2026-07-14-0930-01-pretx.xml" processedAt="2026-09-14T10:00:00.000Z" stage="VALIDATING" summary="XSD validation failed">
  <file>
    <error code="UNSUPPORTED_SCHEMA_VERSION" message="schemaVersion 2.0 is not supported" severity="error"/>
  </file>
  <programmes>
    <programme id="BBC-PROG-00417">
      <error code="OFCOM_GENRE_CODE_UNKNOWN" message="Genre code 99 is not in the Ofcom list" severity="error"/>
    </programme>
  </programmes>
</errorReport>
```

#### Error codes

Error and warning codes are stable strings that a Sender can safely branch on. Three rules apply:

- **The list grows over time.** TEP adds codes as new checks are introduced. Treat an unrecognised error code as a generic failure, and an unrecognised warning code as informational, rather than rejecting the report.
- **XSD faults may arrive as `SCHEMA_ERROR_<n>`.** Where TEP's validator cannot map an XSD violation to a named code it falls back to `SCHEMA_ERROR_` followed by the underlying libxml2 error number. Match on the prefix rather than the whole value.
- **The message carries the detail.** The code identifies the kind of fault; the `message`, `field` and `line` attributes say where it is and what was wrong.

The codes a Sender is most likely to encounter are listed below alongside the rule each relates to. The list is illustrative rather than exhaustive. The rules themselves are documented in the [TEP Ingestion Validation](Diamond2_XML_Field_Reference_and_Implementation_Notes.md#tep-ingestion-validation) section of the Field Reference.

| Code | Reported against | Meaning |
| --- | --- | --- |
| `XML_PARSE_ERROR` | file | The file is not well-formed XML. |
| `SCHEMA_VALIDATION_FAILED` | file | The file failed validation against the Diamond 2 XSD. More specific codes such as `MISSING_REQUIRED_ATTRIBUTE`, `MISSING_REQUIRED_ELEMENT`, `ENUMERATION_VALIDATION_ERROR` and `DATATYPE_VALIDATION_ERROR`, or the `SCHEMA_ERROR_<n>` fallback, identify the individual violations. |
| `UNSUPPORTED_XML_NAMESPACE` | file | The `Document` root element is not in the `urn:cdn:pdx:v1` namespace. |
| `UNSUPPORTED_SCHEMA_VERSION` | file | The `schemaVersion` attribute is not a revision TEP recognises. |
| `PROCESSING_ERROR`, `VALIDATION_TIMEOUT` | file | Processing failed on the TEP side (`severity="critical"`). Resubmit the file unchanged. |
| `DUPLICATE_ID` | record | The same Project ID appears more than once in a Pre-TX file (one update per project per file). |
| `PROJECT_WITHOUT_EPISODE` | record | A non-removal Project contains no Episode. |
| `REQUIRED_FIELD_MISSING` | record | A non-removal record omits an element that is optional in the XSD only so that removal records can validate, such as `ChannelPlatforms` or `PublicationDateTime` on a Publication. |
| `OFCOM_GENRE_MISSING`, `OFCOM_GENRE_MULTIPLE` | record | A Genres container does not contain exactly one Ofcom genre. |
| `OFCOM_SUPER_GENRE_MULTIPLE`, `COMMISSIONER_GENRE_MULTIPLE` | record | A Genres container contains more than one OfcomSuper or more than one Commissioner genre. |
| `OFCOM_GENRE_CODE_UNKNOWN`, `OFCOM_SUPER_CODE_UNKNOWN` | record | A genre or supergenre code is not in the published Ofcom list. |
| `OFCOM_GENRE_SUPER_MISMATCH` | record | The OfcomSuper genre is not the parent of the Ofcom genre. |
| `PUBLICATION_WINDOW_CLOSURE_NOT_ALLOWED_FOR_BROADCAST` | record | `WindowClosureDateTime` was supplied on a Publication with `availabilityMode="broadcast"`. |
| `PUBLICATION_REMOVAL_MISSING_ID` | record | A Publication with `remove="true"` has no `publicationId`. |
| `INVALID_CORE_MIXED_WITH_REGIONAL` | record | A ChannelPlatform combines an `isCore="true"` SubChannel with other SubChannels, or contains more than one `isCore="true"` SubChannel. |
| `FIELD_LOCK_VALUE_DROPPED` | warning | A submitted value was not applied because another party holds a lock on that field (see §6.2). |

### 6.2. Completion Status File

The root element is `<statusReport>`. It carries one `<programme>` element for each record in the file that was written to TEP, with two attributes:

- `id` — the Sender's own Project `id` as it appeared in the file (required). This is deliberately the Sender's identifier rather than TEP's internal one, so that the report can be reconciled against the file that produced it without any lookup.
- `status` — `created` if the record did not previously exist in TEP and was created, or `updated` if it matched an existing record which was updated in place (required).

Four rules govern what appears in the report:

- **A record that wrote nothing gets no `<programme>` element.** A deletion (`remove="true"`) has no created-or-updated answer to give, and neither does a Post-TX publication whose `episodeId` TEP did not recognise, which is skipped. A file in which nothing was written still produces a status file; it simply contains no `<programme>` elements. The status file is the receipt that the file was processed.
- **Skipped publications are visible only here.** A publication with an unrecognised `episodeId` raises no error (see the [Field Reference](Diamond2_XML_Field_Reference_and_Implementation_Notes.md)), so the absence of a `<programme>` element for it in the status file is the only indication a Sender receives that it was not applied.
- **Post-TX files report one element per `<Publication>`.** Each publication is a record, and a publication always updates an existing programme rather than creating one, so every element on a Post-TX status file has `status="updated"`. Several publications against the same programme produce several elements carrying the same `id`. A Sender reconciling by `id` should therefore expect repeats on Post-TX and count elements rather than assume each is a distinct programme.
- **A record that succeeded may still carry warnings.** A record can be accepted and yet have had a value discarded. Such a warning is reported as a `<warning>` element nested under the `<programme>` element concerned.

The commonest warning is a dropped field. A Commissioner can lock a field on a project or episode in TEP, which makes that field read-only to every other party. If an uploaded file supplies a value for a field that another Commissioner has locked, the rest of the record is applied as normal, the locked value is dropped, and a `<warning>` with `code="FIELD_LOCK_VALUE_DROPPED"` is written. A `<warning>` element has the following attributes:

| Attribute | Meaning |
| --- | --- |
| `code` | Stable machine-readable identifier for the kind of warning (required). |
| `message` | Human-readable explanation in English (required). Do not parse it. |
| `field` | Name of the field whose value was dropped, for example `genre` (dropped-field warnings only). |
| `droppedFor` | TEP's identifier for the Commissioner whose value was dropped. For a file sent through this exchange that is the Sender itself (dropped-field warnings only). |
| `projectId` | TEP's internal identifier for the project concerned. This is not the Sender's Project `id`, which is on the enclosing `<programme>` element (dropped-field warnings only). |
| `episodeNumber` | The episode the dropped field belongs to, where the drop was on an episode rather than on the project itself (dropped-field warnings only). |

The Commissioner holding the lock is never identified.

Example:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<statusReport xmlns="urn:tep:pdx:report:1.0" schemaVersion="1.0" file="2026-07-14-0930-01-pretx.xml" processedAt="2026-09-14T10:00:00.000Z">
  <programme id="BBC-PROG-00417" status="created"/>
  <programme id="BBC-PROG-00418" status="updated">
    <warning code="FIELD_LOCK_VALUE_DROPPED" message="Field &quot;genre&quot; was not applied: another party holds this field" field="genre" droppedFor="66f1a2b3c4d5e6f708192a3b" projectId="66f1a2b3c4d5e6f708192a3c"/>
  </programme>
</statusReport>
```

### 6.3. Programme Record Semantics

| Aspect | Rule |
| --- | --- |
| **Identifier** | Sender supplies a unique programme ID per broadcaster. Under the bonnet the Receiver is likely to prepend the broadcaster ID to the programme ID to guarantee global uniqueness (up to 256 chars). |
| **Completeness** | When updating, only changed elements or attributes need to be included. Missing elements will not ***explicitly blank*** out existing values, but empty values will. |
| **Deduplication** | Multiple files may update the same programme; latest file (by lexical order) wins. |

### 6.4. XML Envelope & Versioning

All files carry the `Document` root element in the `urn:cdn:pdx:v1` namespace with a mandatory `schemaVersion` attribute:

```xml
<Document schemaVersion="1.0" xmlns="urn:cdn:pdx:v1">
  <Programmes>
    … Supplier / Project / Episode …
  </Programmes>
</Document>
```

- `schemaVersion` is incremented when breaking changes occur.
- The Receiver validates against the XSD corresponding to the declared version.
