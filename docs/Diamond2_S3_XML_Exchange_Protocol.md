# TV Programme Feed — S3 Exchange Protocol

## Change History

| Version | Date | Change |
| --- | --- | --- |
| 1.1 | 27/11/2025 | Final Word (.docx) revision. |
| 1.2 | 29/7/2026 | Converted from Word to Markdown to simplify change tracking and enable external contributions via GitHub.<br>Corrected the XML envelope example to match the published schemas (`Document` root element, `urn:cdn:pdx:v1` namespace).<br>Fixed section numbering and cross-references.<br>Expanded the filename rules with guidance on naming patterns and processing order. |
| 1.3 | 29/7/2026 | Corrected the bucket layout: live and staging are separate dedicated buckets (not subdirectories of a shared sender root), each with incoming/, complete/ and errors/ at the top level. |

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

- **Prefixing with `PRE_TX` and `POST_TX`.** POST sorts before PRE, so your Post-TX file will be processed first. A publication record referencing an episode ID that TEP has not yet seen is silently ignored (see the [Field Reference](Diamond2_XML_Field_Reference_and_Implementation_Notes.md)), so this will quietly discard transmission records with no error raised anywhere. If you are prefixing this way, it is worth a look at what has actually landed.
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
4. **Completion** —
   - On success ➜ write `complete/<file>.status.xml` (see §6.2).
   - On failure ➜ move the XML to `errors/` and generate `<file>.errors.xml` (see §6.1).

### 6.1. Error Handling

- **All-or-nothing:** if *any* record is invalid, the **entire file** is rejected.
- Error report schema: XML namespace and code list to be published separately.

### 6.2. Completion Status File

- One `<programme>` element per record in the source file.
- `status` attribute values: `created` | `updated`.

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
