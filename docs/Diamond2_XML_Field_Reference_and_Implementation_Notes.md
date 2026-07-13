# Diamond 2 XML Schema: Field Reference and Implementation Notes

## Change History

| Version | Date | Change |
| --- | --- | --- |
| 5.1 | 24/2/2026 | Updated handling of genre at project and episode level. See `/Document/Programmes/Supplier/Project/Genres` |
| 5.2 | 10/3/2026 | Added details of how to delete previously sent project / episode / publication records |
| 5.3 | 12/4/2026 | Added note about the remove attribute not passing XSD validation due to limitation in XSD 1.0 functionality |
| 5.4 | 13/4/2026 | Changed FirstDeliveryDateTarget to just DeliveryDate and updated accompanying explanation |
| 5.5 | 18/5/2026 | Added reference to new web site for hosting schema files and other documentation.<br>Updated details of how Ofcom genres should be encoded.<br>Updated details of how regional variations should be handled. |
| 5.6 | 25/5/2026 | Minor corrections to case errors in SubChannel references and addition of missing @isCore attribute documentation.<br>Updated namespace version number to be consistent with latest XSD files. |
| 5.7 | 26/6/2026 | Modified XSD so that objects with remove=true attribute will now validate correctly, updating documentation to reflect this.<br>Added documentation of /Document/@schemaVersion |
| 5.8 | 27/6/2026 | Added note on behaviour of Post-TX publication ingestion process when an unknown Episode ID is encountered. |
| 5.9 | 9/7/2026 | Converted document from Word to Markdown to simplify change tracking and enable external contributions via GitHub.<br>Corrected the Deleting Records example and removal notes to reflect the minimal XSD-valid removal record (verified with xmllint against the published XSD): enumerated attributes such as availabilityMode must carry a permitted value rather than an empty string. |
| 5.10 | 13/7/2026 | Corrected the Episode slotLength removal notes: a valid ISO 8601 duration value (e.g. PT0S) is still required when removing an Episode, as an empty value will not pass XSD validation, but in the case of deletion the value is ignored (verified with xmllint against the published XSD). |
| 5.11 | 13/7/2026 | Added note that a file may contain multiple Publications containers, as permitted by the XSD. |
| 5.12 | 13/7/2026 | Added note that TEP cannot process multiple updates to the same project within a single Pre-TX file: consolidate to one Project record per project ID per file (updates across separate files are unaffected). Noted that some broadcasters split update streams into one update per file to simplify error handling. |

## Introduction

This document provides detailed descriptions of all fields in the Diamond 2 XML data exchange schema. The XML schema is used to exchange production metadata and transmission data between broadcasters and The Everyone Project (TEP).

Fields marked as Optional may be omitted from the XML. Fields marked as Mandatory must always be included.

## Document Level

### `/Document/@messageID`

**Status:** Optional

**Type:** String

**Description:** Optional identifier for broadcaster's internal use. This field is provided solely for the benefit of the broadcaster creating the file and will not be ingested into TEP or processed by TEP in any way.

### `/Document/@createdAt`

**Status:** Optional

**Type:** DateTime (ISO 8601)

**Description:** Optional timestamp indicating when this XML document was created. As with message ID this is solely for the benefit of the broadcaster creating the file and will not be ingested into TEP or processed by TEP in any way.

### `/Document/@updatedAt`

**Status:** Optional

**Type:** DateTime (ISO 8601)

**Description:** Optional timestamp indicating when this XML document was last updated. As with message ID this is solely for the benefit of the broadcaster creating the file and will not be ingested into TEP or processed by TEP in any way.

### `/Document/@schemaVersion`

**Status:** Mandatory

**Type:** Enumeration

**Permitted values:** `1.0`

**Description:** Required schema revision identifier. TEP uses this value to determine which fields to expect and will reject any unrecognised version. The list of permitted values will be extended as future non-breaking revisions are published. This attribute should currently be set to "1.0" for all documents.

## Pre-TX Data: Programme Metadata

This section covers fields used to describe programme content before transmission or publication.

### Programmes Container

#### `/Document/Programmes/@commissioningDepartment`

**Status:** Optional

**Type:** String

**Description:** Optional commissioning department identifier (e.g., 'Drama', 'Entertainment', 'News'). This is the broadcaster's own name for the department to facilitate reporting within organisations by department. No validation is performed on this field. Like object IDs, this will be stored within TEP in broadcaster-specific namespaces.

### Supplier

The Supplier object identifies the organisation supplying the paperwork for the projects listed. The Supplier specified here will be granted access to add cast and crew details to all projects listed within this supplier object.

#### `/Document/Programmes/Supplier/@id`

**Status:** Mandatory

**Type:** String

**Description:** Broadcaster-specific internal supplier identifier. These IDs will be stored within TEP in broadcaster-specific namespaces so there is no chance of collision if two broadcasters use the same ID for different suppliers. The production company identified by this Supplier.id will be granted access in TEP to administer projects and add cast/crew details.

#### `/Document/Programmes/Supplier/@name`

**Status:** Optional

**Type:** String

**Description:** Optional supplier/production company name. Used solely for reconciling with potential existing supplier accounts in TEP the first time this supplier is presented by a broadcaster. Once the linking process is complete, this attribute will be ignored and changes to this field will have no effect.

#### `/Document/Programmes/Supplier/@companyNumber`

**Status:** Optional

**Type:** String

**Description:** Optional company registration number. Used solely for reconciling with potential existing supplier accounts in TEP. Once linking is complete, this attribute will be ignored.

### Project (Series/Programme)

A Project represents a series, programme, or collection of related episodes. A given project may appear only once per file — see "One Update per Project per File" under "Data Update Behaviour" below.

#### `/Document/Programmes/Supplier/Project/@id`

**Status:** Mandatory

**Type:** String

**Description:** Broadcaster-specific internal project/series identifier.

#### `/Document/Programmes/Supplier/Project/@remove`

**Status:** Optional

**Type:** Boolean

**Description:** Deleting a previously supplied project record is supported by setting this attribute to "true" and sending this along with the project id attribute to identify the project to be deleted. See the "Deleting Records" section under "Data Update Behaviour" below. In normal (not deleting) use, this attribute can be completely omitted. If a project is deleted, then all associated episode records will also automatically be deleted.

#### `/Document/Programmes/Supplier/Project/@name`

**Status:** Mandatory

**Type:** String

**Description:** Project or series name (e.g., 'Line of Duty Series 6').

#### `/Document/Programmes/Supplier/Project/@medium`

**Status:** Mandatory

**Type:** Enumeration

**Permitted values:** `audio` | `video`

**Description:** Primary medium type for this project. Audio is provided for broadcasters who might want to report on radio broadcasts. This can be hard-coded to 'video' for organisations without radio content.

#### `/Document/Programmes/Supplier/Project/@number`

**Status:** Optional

**Type:** String

**Description:** Optional series number (e.g., '6' for Series 6).

#### `/Document/Programmes/Supplier/Project/@numberOfEpisodes`

**Status:** Optional

**Type:** Positive Integer

**Description:** Optional total number of episodes in this project/series.

#### `/Document/Programmes/Supplier/Project/@type`

**Status:** Optional

**Type:** Enumeration

**Permitted values:** `series` | `special` | `feature film`

**Description:** Optional project type classification.

#### `/Document/Programmes/Supplier/Project/GreenlightDate`

**Status:** Optional

**Type:** Date (ISO 8601)

**Description:** Optional greenlight date for the project/series. If omitted, this will be inferred as the date when this project ID was first sent to TEP in an XML file. This field facilitates reporting by greenlight date.

#### `/Document/Programmes/Supplier/Project/DeliveryDate`

**Status:** Optional

**Type:** Date (ISO 8601)

**Description:** Optional target date for when the project as a whole will be delivered. Strictly speaking this should be the date on which the initial version of all episodes in the project will be delivered. If subsequent versions of episodes are created this should not affect this date.

This need only be the best guess of the target date at time of submission. This should be provided if available but omitted if not. Production companies will be asked to complete this if it is not supplied by the broadcaster. **IMPORTANT:** Broadcasters who do not have this data should completely omit this object rather than include an empty element. An empty element would cause any date provided by the production company to be blanked out when the project data is passed in the XML feed.

#### `/Document/Programmes/Supplier/Project/Genres`

**Status:** Optional

**Type:** Container

**Description:** Optional container element for genre classifications at project level. If present this must contain exactly one Ofcom genre, optionally one OfcomSuper genre, and optionally one Commissioner genre. This project level genre will be stored against the project. Any episode without any genre will inherit this project level genre.

When the first episode for a project is encountered, then if at this point the project has no genre data, then the genre data for that episode will be used to populate the project level genre details.

#### `/Document/Programmes/Supplier/Project/Episode/Genres`

**Status:** Optional

**Type:** Container

**Description:** A Genres object can optionally be supplied at the episode level to override the project-wide genre settings as described above.

#### `/Document/Programmes/Supplier/Project/Genres/Genre`

#### `/Document/Programmes/Supplier/Project/Episode/Genres/Genre`

**Type:** Container

**Content:** Genre name (String)

**Description:** Individual genre element with a type attribute and text content containing the genre name. The type attribute must be one of: Ofcom, OfcomSuper, or Commissioner.

#### `/Document/Programmes/Supplier/Project/Genres/Genre/@type`

#### `/Document/Programmes/Supplier/Project/Episode/Genres/Genre/@type`

**Status:** Optional

**Type:** Enumeration

**Permitted values:** `Ofcom` | `OfcomSuper` | `Commissioner`

**Description:** The type of Genre being specified

**Genre Constraints:**

- **Ofcom genre:** Mandatory - exactly one required. Ofcom genres must be provided using the short code (e.g. "CHDRA") rather than the long description (e.g. "Children's drama"). The Ofcom genre value will be validated against Ofcom's official genre code list. At the time of the last update of this document the list of Ofcom genre codes was as follows:

  `ANIMATION, ARTINF, ARTPERF, CAFF, CHDRA, CHENT, CHILDANIM, CHINF, CHNEWS, CLASSMUS, CLOSE, COMOTH, CONTMUS, DOC, DRDOC, DROTH, EDOTH, ENOTH, FACTENT, FILM, GENFAC, GSOTHER, GSPRIZ, HISTORY, HOBLEISURE, HOMESHOP, LONG, NATENV, NEWS, NEWS24, NEWSNT, PARLCAFF, PARLNEWS, PPB, PRESCL, RELFAITH, RELSEV, SCHOOLS, SCIMEDTEC, SITCOM, SOCACT, SPECEVT, SPEVT, SPOTH, TALKMAG, WEATHER`

- **OfcomSuper genre:** Optional - maximum one allowed. If omitted, will be automatically derived from the Ofcom genre (every Ofcom genre is a child of a pre-defined supergenre parent). Once again this should be provided in the short code form. At the time of the last update of this document the list of Ofcom genre codes was as follows:

  `ARTS, CHILDREN, COMEDY, CRTAFFAIR, DRAMA, EDUCATION, ENT, FACTUAL, FACTUALENT, FTRFILMS, LEISURE, MISC, MUSIC, NEWSWTH, RELIGION, SPORT`

- **Commissioner genre:** Optional - maximum one allowed. Broadcaster-specific genre classification with no validation, allowing broadcasters to use their own internal genre taxonomies. If Commissioner genre needs to be negated at a per-episode level (i.e. the project has a commissioner genre default, but one particular episode needs to completely omit the commissioner genre) then an empty `<Genre type="Commissioner" />` should be included within that episode.

### Episode

#### `/Document/Programmes/Supplier/Project/Episode`

**Status:** Mandatory (except for Project removal)

**Type:** Object

**Description:** An Episode represents an individual programme or episode within a project/series. Every Project object must have at least one Episode object unless the Project object has the `remove="true"` attribute set, in which case Episode is not required. However, this is not reflected in the XSD since XSD 1.0 cannot cope with this sort of conditional structure definition. The requirement for projects to have at least one Episode is validated by TEP on data ingestion, so whilst a non-removal Project without an Episode subobject will pass XSD validation, it will be rejected by TEP on ingestion.

#### `/Document/Programmes/Supplier/Project/Episode/@id`

**Status:** Mandatory

**Type:** String

**Description:** Broadcaster-specific internal episode identifier. This is referenced later in the Post-TX xml file. This ID must be unique across all projects.

#### `/Document/Programmes/Supplier/Project/Episode/@remove`

**Status:** Optional

**Type:** Boolean

**Description:** Deleting a previously supplied episode record is supported by setting this attribute to "true" and sending this along with the episode id attribute to identify the episode to be deleted. See the "Deleting Records" section under "Data Update Behaviour" below. In normal (not deleting) use, this attribute can be completely omitted

#### `/Document/Programmes/Supplier/Project/Episode/@name`

**Status:** Mandatory

**Type:** String

**Description:** Episode name or title. Supports production company ease of work. When removing an Episode (i.e. `remove="true"`) the name attribute remains mandatory but the contents will be ignored so a hard coded empty or arbitrary value can be used.

#### `/Document/Programmes/Supplier/Project/Episode/@slotLength`

**Status:** Mandatory

**Type:** Duration (ISO 8601)

**Example:** `PT1H` (1 hour), `PT30M` (30 minutes), `PT1H30M` (1 hour 30 minutes)

**Description:** Slot length or duration in ISO 8601 format. The precise duration is not essential here; broadcast slot duration is acceptable. If the duration varies between different versions of the episode, provide a duration that is representative of most versions. When removing an Episode (i.e. `remove="true"`) the slotLength attribute remains mandatory and a valid ISO 8601 duration value is still required to pass XSD validation (it cannot be left empty, so use a hard coded value such as `PT0S`), but in the case of deletion the value will be ignored.

#### `/Document/Programmes/Supplier/Project/Episode/@number`

**Status:** Optional

**Type:** String

**Description:** Optional episode number within the series. Note that this is not the broadcaster's internal episode identifier (that's Episode.id), but rather the episode's position in the series.

#### `/Document/Programmes/Supplier/Project/Episode/@makerId`

**Status:** Optional

**Type:** String

**Description:** Optional supplier ID of the production company creating the content. Used when paperwork completion is outsourced to a different company than the content maker. The parent Supplier object identifies who is completing the paperwork (and who gets access to administer the project in TEP), whilst this field identifies who is actually making the content. This field is for reporting purposes only and does not grant the referenced production company any access to administer the project in TEP. If omitted, defaults to the parent Supplier.id.

#### `/Document/Programmes/Supplier/Project/Episode/Tags`

**Status:** Optional

**Type:** Container

**Description:** Optional container for arbitrary key-value tags. Tags are provided to support reporting by broadcasters on arbitrary attributes not captured elsewhere in the XML. Tags can be added at any time and can be retro-fitted to previously supplied episodes by re-sending updated episode records.

#### `/Document/Programmes/Supplier/Project/Episode/Tags/Tag`

**Attributes:** `@name` (String, mandatory)

**Content:** String value

**Example:** `<Tag name="series name">Line of Duty</Tag>`

**Description:** Individual tag with a name attribute and text value. To be most effective for production companies, include colloquial names for series.

#### `/Document/Programmes/Supplier/Project/Episode/ReleaseDate`

**Status:** Optional

**Type:** Date (ISO 8601)

**Description:** Optional first release date. This is when the episode was first ever released (e.g., a film may be released first in cinemas). If omitted, this will be inferred as the earliest publicationDateTime encountered for this episode in the Post-TX data feed.

#### `/Document/Programmes/Supplier/Project/Episode/Genres`

**Status:** Optional

**Type:** Container

**Description:** Optional genres container at episode level. If provided, overrides the project-level genres for this specific episode. If omitted, the episode inherits the genres from its parent Project. Must include exactly one Ofcom genre if present. OfcomSuper genre is optional (derived from Ofcom if absent). Commissioner genre is optional (maximum one allowed).

## Post-TX Data: Publication & Transmission

This section covers fields used to describe transmission events and on-demand availability windows after content has been published.

### Publications Container

As per the XSD, a single file may contain multiple `<Publications>` containers under `<Document>`, each holding one or more `<Publication>` objects. There is no particular benefit to doing this: Publication objects do not inherit anything from their parent Publications container, so the effect is the same either way. It is up to the sender whether to place many Publication objects in one Publications envelope, or to send many Publications containers each with just one Publication.

### Publication

A Publication represents either a broadcast transmission or an on-demand availability window. If a programme is both broadcast and available on-demand, there will be two separate Publication records.

#### `/Document/Publications/Publication/@episodeId`

**Status:** Mandatory

**Type:** String

**Description:** Mandatory reference to the episode ID from the Pre-TX metadata. Links this publication event to a specific episode. When removing a Publication record (i.e. `remove="true"`) the episodeId attribute remains mandatory but the contents will be ignored, so a hard coded empty string (`episodeId=""`) or arbitrary value can be used.

**Note 1:** If your transmission tracking system uses version IDs that consist of the episode ID with a suffix (e.g., episode123/01, episode123/02), you should remove the version suffix to provide just the episode ID (e.g., episode123).

**Note 2:** If an unknown episodeId is provided this publication record will silently be ignored. This is intentional. The reasoning for this is that the transmission systems delivering publication data in the Post-TX stream are often quite separate from the commissioning systems that are the source of the Pre-TX data. There are many situations where some content may not exist in the commissioning system, or may exist, but may not have been included in the Pre-TX data feed (e.g. if that content is not being monitored in Diamond). In most cases the transmission logging system will not know if the programme has been sent to Diamond or not, thus filtering out from the transmission data feed episodes that were not included in the Pre-TX data feed will be difficult or impossible. The decision was taken therefore to allow the transmission system to pass publication records for all episodes regardless of whether they were included in the Pre-TX feed, and simply ignore those records where the episode ID is not recognised. Obviously this means that great care needs to be taken to get the episode ID correct, since if it is wrong and thus fails to join up to the intended episode, then no error will be flagged.

**Note 3:** A side effect of Note 2 is that the episode must have been presented in the Pre-TX XML feed, before associated transmission records are presented in the Post-TX publication feed, otherwise the publication record will be silently ignored.

#### `/Document/Publications/Publication/@availabilityMode`

**Status:** Mandatory

**Type:** Enumeration

**Permitted values:** `broadcast` | `onDemand`

**Description:** Differentiates broadcasts from on-demand windows. Each publication record refers either to a transmission or an on-demand availability window. When removing a Publication record (i.e. `remove="true"`) the availabilityMode attribute remains mandatory and must be set to one of the permitted values ('broadcast' or 'onDemand') to pass XSD validation; the value supplied will be ignored.

#### `/Document/Publications/Publication/@publicationId`

**Status:** Optional (except when `remove="true"`)

**Type:** String

**Description:** Optional broadcaster's unique ID for this publication event. Allows corrections to publication event data. Can be omitted if there is no future intention for your publication event data to be edited or corrected. If removing a Publication record (i.e. `remove="true"`) this attribute must be present.

#### `/Document/Publications/Publication/@remove`

**Status:** Optional

**Type:** Boolean

**Description:** Deleting a previously supplied publication record is supported by setting this attribute to "true" and sending this along with the publication id attribute to identify the publication to be deleted. See the "Deleting Records" section under "Data Update Behaviour" below. In normal (not deleting) use, this attribute can be completely omitted

#### `/Document/Publications/Publication/@isRepeat`

**Status:** Optional

**Type:** Boolean

**Permitted values:** `true` | `false`

**Description:** Optional boolean indicating whether this is a repeat transmission/publication. Use false for first transmission/publication, true for repeats. If omitted, the first publication record received will be automatically treated as first (isRepeat=false). **IMPORTANT:** If your transmission records may be sent out of chronological order (i.e., a later transmission sent before an earlier one), do not omit this attribute, as the first record received will be marked as first transmission regardless of its actual transmission date.

#### `/Document/Publications/Publication/@isPrimary`

**Status:** Optional

**Type:** Boolean

**Permitted values:** `true` | `false`

**Description:** Optional boolean indicating whether this is the primary transmission/publication for reporting purposes. The primary publication is typically the most important broadcast (e.g., peak-time transmission rather than an overnight repeat). If omitted, the first publication that is treated as first (isRepeat=false or inferred as such) will be automatically marked as primary (isPrimary=true).

#### `/Document/Publications/Publication/ChannelPlatforms`

**Status:** Mandatory (except for Publication removal)

**Type:** Container

**Description:** Container for one or more ChannelPlatform elements. Multiple ChannelPlatform elements can be included if the publication was broadcast on multiple channels simultaneously. The ChannelPlatforms object is mandatory unless the Publication has the `remove="true"` attribute set. XSD 1.0 does not support conditional validation, so the XSD file shows the ChannelPlatforms as being optional in order to allow Publication removal XML to be validated against the XSD. However any non-removal Publication record without a ChannelPlatforms object will be rejected by TEP as invalid on ingestion.

#### `/Document/Publications/Publication/ChannelPlatforms/ChannelPlatform/@label`

**Status:** Mandatory

**Type:** String

**Description:** Human-readable channel or platform name (e.g., 'BBC One', 'iPlayer', 'ITV1').

#### `/Document/Publications/Publication/ChannelPlatforms/ChannelPlatform/@id`

**Status:** Optional

**Type:** String

**Description:** Channel identifier. If omitted, defaults to the value of the label attribute.

#### `/Document/Publications/Publication/ChannelPlatforms/ChannelPlatform/SubChannel`

**Status:** Mandatory

**Type:** Container

**Description:** Represents a sub-channel or regional variant of the parent channel. Multiple SubChannel elements can be included if the publication was broadcast on multiple sub-channels simultaneously (e.g., broadcast on both BBC One Wales and BBC One Scotland at the same time). For more details see section on Regional Variations below.

#### `/Document/Publications/Publication/ChannelPlatforms/ChannelPlatform/SubChannel/@label`

**Status:** Mandatory

**Type:** String

**Description:** Human-readable sub-channel label (e.g., 'Wales', 'Scotland', 'HD').

#### `/Document/Publications/Publication/ChannelPlatforms/ChannelPlatform/SubChannel/@id`

**Status:** Optional

**Type:** String

**Description:** Sub-channel identifier. If omitted, defaults to the value of the label attribute.

#### `/Document/Publications/Publication/ChannelPlatforms/ChannelPlatform/SubChannel/@isCore`

**Status:** Optional

**Type:** Boolean

**Permitted values:** `true` | `false`

**Description:** Indicates whether this SubChannel represents the core network feed (i.e., the content that goes out across the whole network rather than a regional opt-out). Set to "true" for the core network feed; omit or set to "false" for regional sub-channels. At most one SubChannel per ChannelPlatform may carry isCore="true". This constraint cannot be expressed in XSD 1.0 and is enforced by TEP during ingestion.

Broadcasters who do not operate any regional sub-channels should include a single SubChannel with isCore="true" on every Publication, using whatever id and label values are most appropriate (typically a single fixed value hard-coded into the generation pipeline).

Broadcasters who operate an entirely regional service with no national feed would never use isCore="true", sending only regional SubChannel records.

See the "Regional Variations" section for detailed examples and guidance on valid/invalid combinations.

#### `/Document/Publications/Publication/PublicationDateTime`

**Status:** Mandatory (except for Publication removal)

**Type:** DateTime (ISO 8601)

**Description:** Transmission date/time for broadcast content, or the start of the availability window for on-demand content. The PublicationDateTime object is mandatory unless the Publication has the `remove="true"` attribute set. XSD 1.0 does not support conditional validation, so the XSD file shows the PublicationDateTime as being optional in order to allow Publication removal XML to be validated against the XSD. However any non-removal Publication record without a PublicationDateTime will be rejected by TEP as invalid on ingestion.

#### `/Document/Publications/Publication/WindowClosureDateTime`

**Status:** Optional

**Type:** DateTime (ISO 8601)

**Description:** End of the availability window for on-demand content. Valid only when availabilityMode='onDemand'. If omitted, interpreted as indefinite availability. The closure date may be uncertain at time of initial publication; in this case, it can be omitted initially and later added by resubmitting with the same publicationId.

## Data Update Behaviour

### Record Identification

All ID fields (Supplier.id, Project.id, Episode.id) are internal identifiers defined by the broadcaster. These will be stored within TEP in broadcaster-specific namespaces to prevent collisions if two broadcasters use the same ID for different records.

### Creating and Updating Records

When an XML file is processed, the system uses the supplied IDs to determine whether to create a new record or update an existing one. If no record with the supplied ID exists, a new record will be created in TEP. If a record with the supplied ID already exists, the data in the XML will overwrite the previously stored data for that record. Records can be resent as many times as needed to update information.

### One Update per Project per File

TEP's ingestion process cannot process multiple updates to the same project within a single Pre-TX file. Each project ID must therefore appear at most once per file: if your systems generate several changes to the same project (or to episodes within it), consolidate them into a single Project record presenting one complete view of the project and its episodes before sending. This restriction cannot be expressed in the XSD, so a file containing more than one Project record with the same ID will pass XSD validation but will be rejected by TEP on ingestion.

This limitation applies only within a single file. Updating the same project repeatedly across separate files is fully supported and is the normal way project data evolves over time.

Some broadcasters have found it useful to split an update stream (for example, a queue of updates generated throughout the day) into separate XML files, each containing just one project — or even just one update action — per file. This results in more files, but the one-update-per-project-per-file limitation becomes a non-issue and error handling is much easier. Ingestion is all-or-nothing: an error anywhere in a file causes the whole file to be rejected, which otherwise requires the sender to parse the error report to work out which of potentially hundreds of updates caused the problem. With one update per file, a rejected file identifies the problem update exactly. Broadcasters are free to take whichever approach they prefer — many updates per file, or one per file — this is simply a suggestion of one potential benefit of the latter.

### Deleting Records

Whilst updating an existing record is achieved by resending it with the same ID and revised data, some record types support explicit deletion via the remove attribute.

When `remove="true"` is set on a supported object, TEP will delete the identified record. Only the ID attribute is used to identify the object, but the XSD still requires the object's other mandatory attributes to be present, so they must be supplied with placeholder values. Their contents will be ignored by TEP. Attributes typed as plain strings may be left empty, attributes with an enumerated type (such as `availabilityMode`) must be given one of their permitted values, and attributes with other typed content (such as `slotLength`, a duration) must be given a syntactically valid value (e.g. `PT0S`) to pass XSD validation — but in the case of deletion all these values will be ignored. Child elements should be omitted entirely.

The following is the minimal removal record for a previously submitted Publication, verified against the published XSD:

```xml
<Publications>
  <Publication
    publicationId="pub-456"
    episodeId=""
    availabilityMode="broadcast"
    remove="true" />
</Publications>
```

Deletions and new records can be sent in the same XML file.

The remove attribute is only supported on the following objects:

- `/Document/Programmes/Supplier/Project`
- `/Document/Programmes/Supplier/Project/Episode`
- `/Document/Publications/Publication`

### Optional Field Behaviour

The system treats omitted optional fields differently from empty optional fields. If an optional field was supplied in a previous file but is omitted in a later update of the same record, the previously stored data will be retained. This allows you to send partial updates without having to resend all optional fields every time.

However, if you want to remove previously stored data for an optional field, you must provide an empty element or attribute (not simply omit it). This explicit approach prevents accidental data loss whilst allowing intentional data removal.

### Partial Updates

If a single field has changed, the entire record should be re-posted including all other mandatory fields, even if those fields have not changed. Optional fields need not be resent if they haven't changed, although including them will not cause any issues.

## Regional Variations

Several broadcasters operate channels with regional or sub-brand variants (e.g. BBC One Wales, BBC One Scotland; ITV regions). Diamond 2 needs to distinguish core network output from regional opt-outs to support CDN's reporting requirements, which include core-network-only reporting, regional-only reporting, user-experience reporting for a given region, and full-channel reporting across the network and all SubChannels.

### The "Core Plus Variations" Model

Diamond 2 uses a "core plus variations" model. A single Publication record represents the broadcast on the core network feed. Where a regional variation overrides the core feed in a specific region, an additional Publication record is sent for that variation. Both records carry the same `<ChannelPlatform>` but differ in their `<SubChannel>`.

Reporting then follows directly from this:

- Core network only: publications where `<SubChannel>` is marked as the core feed.
- Regional only (region X): publications where `<SubChannel>` is X.
- User experience for region X: publications where `<SubChannel>` is either the core feed or X.
- All content: all publications for the channel.

### Indicating The Core Feed: The isCore Attribute

`<SubChannel>` is mandatory on every `<ChannelPlatform>`. To indicate that a particular subchannel represents the core network feed, the isCore="true" attribute is set on that `<SubChannel>`. Regional subchannels omit the attribute (or set it to false).

Within any single `<ChannelPlatform>`, at most one `<SubChannel>` may carry isCore="true".

Broadcasters who do not operate any regional subchannels should include a single hard coded `<SubChannel>` with isCore="true" on every Publication. The id and label values can be whatever is most appropriate for the broadcaster, typically a single fixed value hard-coded into the generation pipeline. E.g.

```xml
<SubChannel label="Network" id="network" isCore="true"/>
```

Broadcasters who do operate subchannels mark their core network SubChannel with isCore="true" and identify any regional SubChannels without the attribute.

The "core network" concept is therefore available to broadcasters who have one, and unobtrusive to those who don't. A broadcaster with no national feed at all (e.g. an entirely regional service) would simply never use isCore="true", sending only regional `<SubChannel>` records.

### Invalid Combinations

A single `<ChannelPlatform>` that contains both an isCore="true" `<SubChannel>` and one or more other `<SubChannel>` elements is semantically invalid. Either the content went out on the core network feed (which by definition spans all regions unless overridden) or it went out on a specific subset of subchannels. The two cannot be combined in one record.

Similarly, a `<ChannelPlatform>` cannot contain more than one `<SubChannel>` with isCore="true".

These constraints cannot be expressed in XSD 1.0, so they are enforced at TEP's ingestion layer. Affected records will be rejected and the error reported back via the standard error feedback channel.

### Examples

A BBC One transmission of a national news bulletin at 22:00, broadcast across the entire UK:

```xml
<Publication episodeId="news-bulletin-2200" availabilityMode="broadcast">
  <PublicationDateTime>2026-05-14T22:00:00Z</PublicationDateTime>
  <ChannelPlatforms>
    <ChannelPlatform label="BBC One" id="bbc_one">
      <SubChannel label="Network" id="network" isCore="true"/>
    </ChannelPlatform>
  </ChannelPlatforms>
</Publication>
```

A BBC One Wales rugby opt-out replacing that bulletin in Wales only at the same slot:

```xml
<Publication episodeId="rugby-coverage-live" availabilityMode="broadcast">
  <PublicationDateTime>2026-05-14T22:00:00Z</PublicationDateTime>
  <ChannelPlatforms>
    <ChannelPlatform label="BBC One" id="bbc_one">
      <SubChannel label="Wales" id="bbc_one_wales"/>
    </ChannelPlatform>
  </ChannelPlatforms>
</Publication>
```

A broadcaster with no regional SubChannels at all uses a single core-marked SubChannel everywhere:

```xml
<Publication episodeId="episode789" availabilityMode="broadcast">
  <PublicationDateTime>2026-05-14T21:00:00Z</PublicationDateTime>
  <ChannelPlatforms>
    <ChannelPlatform label="Channel 5" id="channel_5">
      <SubChannel label="Network" id="network" isCore="true"/>
    </ChannelPlatform>
  </ChannelPlatforms>
</Publication>
```

## Processing Limits and Recommendations

### System Capacity

The system will process up to 1,000 files and 10,000 episodes per day (whichever limit is reached first). If these limits are exceeded, excess records will be buffered in the S3 bucket. There will be no loss of data, but there may be a delay in working through the backlog of files.

### File Structure Flexibility

The XML structure is flexible to accommodate different organisational approaches. You can include multiple records in a single file, or provide separate records in separate files. This flexibility extends to sub-objects: one Programmes object can contain multiple Supplier objects or just one; one Project can contain multiple Episodes, or Episodes can be sent as completely separate records in separate files.

### Update Frequency

Updates can be sent whenever convenient for your organisation. These could be event-driven (triggered by internal data changes) or batched (for example, as a daily digest). The sooner data is sent, the sooner broadcasters and production companies can begin to pull statistics for a production.

### Data Quality Expectations

It is anticipated that for some fields, the data might be refined over time. This is not an issue, and the general preference is to receive data as soon as possible, even if there is a chance that it might be later superseded or corrected. This iterative approach to data quality is acceptable provided that two conditions are met: firstly, the data should ultimately settle to the most accurate version possible; and secondly, broadcasters should understand that where data is subject to change, any reports built from that data will similarly be subject to corresponding changes.

## Schema Information

**Namespace:** `urn:cdn:pdx:v1`

**Schema Version:** 1.0

**Date:** May 2026

XSD schema files and example XML documents are available from: https://schemas.creativediversitynetwork.com/
