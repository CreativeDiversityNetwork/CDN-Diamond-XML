# Diamond 2 Programme Data Exchange (PDX)

XML schemas and documentation for the **Diamond 2** programme data exchange protocol, maintained by [Creative Diversity Network](https://creativediversitynetwork.com/) (CDN).

Diamond 2 is the next generation of CDN's diversity monitoring system for the UK broadcasting industry. The PDX schema defines how broadcasters exchange production metadata and transmission data with [TEP (The Everyone Project)](https://theeveryoneproject.com/) — the platform that collects and reports on diversity data across the industry.

This repository is published for Diamond 2 participants and for anyone in the wider broadcasting industry who may find the schema useful as a reference or starting point.

## Overview

The exchange uses two types of XML document:

- **PRE-TX (Programmes)** — Production metadata sent before transmission: projects, episodes, genres, suppliers, and scheduling information.
- **POST-TX (Publications)** — Transmission records sent after broadcast or on-demand publication: channels, platforms, availability windows, and repeat/primary status.

Each document type has its own standalone XSD schema. PRE-TX and POST-TX data are sent as separate XML files.

## Documentation and schemas

| File | Description |
|------|-------------|
| [Diamond2_PreTX_v1.1.xsd](Diamond2_PreTX_v1.1.xsd) | XSD schema for PRE-TX (Programmes) documents |
| [Diamond2_PostTX_v1.1.xsd](Diamond2_PostTX_v1.1.xsd) | XSD schema for POST-TX (Publications) documents |
| [Diamond2_PreTX_Example.xml](Diamond2_PreTX_Example.xml) | Example PRE-TX document |
| [Diamond2_PostTX_Example.xml](Diamond2_PostTX_Example.xml) | Example POST-TX document |
| [Diamond2_XML_Field_Reference_and_Implementation_Notes.docx](Diamond2_XML_Field_Reference_and_Implementation_Notes.docx) | Field-by-field reference and implementation guidance |
| [Diamond2_S3_XML_Exchange_Protocol.docx](Diamond2_S3_XML_Exchange_Protocol.docx) | S3-based file exchange protocol specification |
| [Diamond2_S3_Authentication.docx](Diamond2_S3_Authentication.docx) | AWS authentication setup guide for S3 upload access |

## Schema structure

Both schemas share the namespace `urn:cdn:pdx:1.0` and use XSD 1.0. The root element is always `<Document>`.

### PRE-TX hierarchy

```
Document
  └── Programmes
        └── Supplier
              └── Project
                    ├── GreenlightDate
                    ├── DeliveryDate
                    ├── Genres
                    │     └── Genre (@type: Ofcom | OfcomSuper | Commissioner)
                    └── Episode
                          ├── Tags → Tag
                          ├── ReleaseDate
                          └── Genres (optional override of project-level genres)
```

### POST-TX hierarchy

```
Document
  └── Publications
        └── Publication
              ├── PublicationDateTime
              ├── WindowClosureDateTime (on-demand only)
              └── ChannelPlatforms
                    └── ChannelPlatform
                          └── SubChannel (regional variants)
```

## Validating XML

Validate example files against the schemas using `xmllint`:

```bash
xmllint --schema docs/Diamond2_PreTX_v1.1.xsd --noout docs/Diamond2_PreTX_Example.xml
xmllint --schema docs/Diamond2_PostTX_v1.1.xsd --noout docs/Diamond2_PostTX_Example.xml
```

## XSD 1.0 limitations

Several business rules cannot be expressed in XSD 1.0 and are enforced server-side by TEP during ingestion:

- **Genre count constraints** — exactly one Ofcom genre required; at most one OfcomSuper; at most one Commissioner.
- **Conditional requirements for deletion** — when `remove="true"` is set on a Project, Episode, or Publication, only the identifying ID attribute is required. The schema cannot express "if remove is true, other fields are optional." Files containing removal records will not pass schema validation; this is expected.
- **WindowClosureDateTime** — only valid when `availabilityMode="onDemand"`.

## Delivery protocol

XML files are exchanged via Amazon S3. Each broadcaster has a dedicated S3 bucket with the following directory structure:

```
<sender-root>/
  ├── live/
  │   ├── incoming/    # broadcaster uploads XML files here
  │   ├── complete/    # status files for successfully ingested XML
  │   └── errors/      # rejected files with error reports
  └── staging/         # same layout, for testing
```

Files in `incoming/` are processed in lexical filename order. The entire file is accepted or rejected as a unit — there is no partial ingestion. Status files and error reports are retained for 7 days.

See [Diamond2_S3_XML_Exchange_Protocol.docx](Diamond2_S3_XML_Exchange_Protocol.docx) and [Diamond2_S3_Authentication.docx](Diamond2_S3_Authentication.docx) for full details on the delivery mechanism and AWS authentication setup.

## Data update behaviour

- **Create or update** — if a record with the given ID exists, it is updated; otherwise a new record is created.
- **Partial updates** — mandatory fields must always be included, but unchanged optional fields can be omitted and their existing values will be retained.
- **Deletion** — set `remove="true"` on a Project, Episode, or Publication along with its ID to delete a previously submitted record.
- **Genre inheritance** — episodes without genres inherit from their parent project. Episode-level genres override the project-level genres entirely.

## License

This project is licensed under the [Apache License 2.0](https://github.com/CreativeDiversityNetwork/CDN-Diamond-XML/blob/main/LICENSE).

Copyright 2025 Creative Diversity Network.
