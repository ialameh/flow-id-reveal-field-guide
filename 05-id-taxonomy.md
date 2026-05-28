# 05. Salesforce Id taxonomy

The extension classifies every Id-shaped token in scanned content by prefix. This chapter is the reference table plus the rules the classifier applies.

## The format

Salesforce sObject record Ids are 15 or 18 characters of `[A-Za-z0-9]`.

- **15 char Ids** are case-sensitive and unique within their object type.
- **18 char Ids** add a 3-char checksum suffix that makes the Id case-insensitive while preserving uniqueness. Most REST APIs return 18 char.

Both forms are valid. The extension regex matches both:

```js
/[0-9a-zA-Z]{15,18}/g
```

## The three-character prefix

The first three characters encode the **object type**. Salesforce maintains a registry of these. Every sObject ever defined has a prefix. The extension ships with a curated dictionary of the prefixes most likely to appear in flow metadata.

## Curated prefix dictionary (v0.5.2)

| Prefix | Object | Notes |
|---|---|---|
| `001` | Account | |
| `003` | Contact | |
| `005` | User | |
| `006` | Opportunity | |
| `00D` | Organization | The org's own Id |
| `00Q` | Lead | |
| `00T` | Task | |
| `00U` | Event | |
| `00e` | Profile | |
| `01p` | ApexClass | |
| `01q` | ApexTrigger | |
| `0A8` | ApexClass | Legacy prefix |
| `0AB` | FlowInterview | Runtime flow execution |
| `0Af` | DeployRequest | MDAPI deploy job |
| `0DM` | Group | |
| `0OD` | DataModelObject | Data Cloud |
| `0OE` | DataLakeObject | Data Cloud |
| `0PS` | PermissionSet | |
| `0Ps` | PermissionSetGroup | |
| `0Pj` | PartyConsent | Marketing Cloud Growth |
| `0VK` | ManagedContentChannel | CMS |
| `0VP` | ContactPointConsent | Marketing Cloud Growth |
| `0WW` | ManagedContent | CMS |
| `0Xl` | CommSubscription | Marketing Cloud Growth |
| `0eB` | CommSubscriptionChannelType | Marketing Cloud Growth |
| `0eF` | EngagementChannelType | Marketing Cloud Growth |
| `300` | FlowDefinition | Container for all versions |
| `301` | Flow | Specific version |
| `500` | Case | |
| `Mzc` | ManagedContentSpace | CMS workspace |

To add a prefix: edit `src/utils/id-patterns.js` and the mirrored constant in `src/content.js`. Keep both in sync.

## Why some Ids are not in the dictionary

The dictionary is intentionally narrow. Many Id prefixes exist in Salesforce but rarely appear in flow metadata:

- Metadata API types (FlowDefinition is `300`, but you rarely see flow Ids referencing other flow definitions)
- Setup-only objects (FieldDefinition, RecordTypeSettings)
- Audit objects (LoginHistory, SetupAuditTrail)

The extension's "Hide unknown prefixes" checkbox (on by default in the Scan tab) suppresses Ids whose prefix is not in the dictionary. Unticking it reveals everything the regex matched. Most uncategorised matches are false positives (Aura framework Ids, internal Lightning component Ids).

## The classifier rules

`looksLikeSfId(s)` returns true if all of:

1. `s.length === 15 || s.length === 18`
2. `/^[0-9a-zA-Z]+$/.test(s)` - pure alphanumeric
3. `/\d/.test(s.slice(0, 5))` - at least one digit in the first 5 characters

Rule 3 is a heuristic to filter false positives from camelCase identifier names. A real SF Id has the pod code in characters 3-5, which is always alphanumeric and almost always includes digits. A camelCase JavaScript identifier like `engagementChannelT` has no digits in its first 5 chars and is correctly rejected.

`classify(s)` extracts the prefix and looks it up in the dictionary. If found, returns `{ id, prefix, length, object, known: true }`. Otherwise `known: false`.

The scan filters to `known: true` by default.

## Pod codes (positions 3-5)

The middle two characters (positions 3-4, 0-indexed) are the **pod code**. They identify the database instance the org runs on.

Examples from real orgs:

- `WG` - SilverChef Omni sandbox
- `WF` - A SilverChef Omni's UAT sibling sandbox
- `5g` - Issam's personal Dev Hub (00D5g00000KBtCUEA1)

Two records on the same pod always start with the same two pod-code characters. Two records on different pods do not.

This is why cross-org Id comparison is straightforward: if pod codes match, the records came from the same database originally (sandbox refreshed from same prod). If pod codes differ, you are looking at two unrelated orgs and Ids never agree.

## The 18-char suffix

The last 3 characters of an 18-char Id are a checksum over the first 15 characters. The algorithm: for each block of 5 chars, build a 5-bit mask of which chars are uppercase, treat as a number, map to base-32 character. Three blocks → three checksum chars.

The extension does not compute or verify the checksum. The regex accepts any 15 or 18 char alphanumeric string. Salesforce rejects invalid checksums when you try to use them, which is good enough.

## What CMS content keys are (and are not)

The `<form>` reference in a form-triggered flow looks like:

```
marketing--Default_Content_Workspace.sfdc_cms__formHandler--MCP4JHBDDOEBHHJKMUYCEKF4YHGY
```

The trailing `MCP4JHBDDOEBHHJKMUYCEKF4YHGY` is **not** a sObject Id. It is:

- 28 characters (not 15 or 18)
- Mostly uppercase letters with some digits
- Generated by the CMS publishing pipeline, not by the sObject Id sequencer
- Unique within the workspace
- Org-specific

The extension's classifier does NOT match CMS content keys (the length is wrong, the regex caps at 18). The CMS Form helper in the Inspect tab handles these separately, using a different parser that splits the form reference on `.` and `--`.

See [Chapter 15](15-cms-form-helper.md) for the full CMS content key handling.

## Custom Metadata Type records

Records inside a Custom Metadata Type (anything ending `__mdt`) have their own prefixes. The dictionary does not include CMT prefixes by default because they vary per CMT definition. If you have a heavily CMT-driven flow, add the prefixes you care about to the dictionary.

To find a CMT's prefix: query `SELECT KeyPrefix FROM EntityDefinition WHERE QualifiedApiName = '<Type>__mdt'`.

## Other Id-shaped tokens that are not Ids

False positives the heuristic intentionally accepts (and that the classifier then rejects):

- 18-char hashes / GUIDs that happen to be alphanumeric → fail dictionary lookup
- Tooling API Aura component refs (`window:auraTokens`) → fail dictionary lookup
- Flow node API names (`Email_Consent`) → fail length check (under 15)

Sticky false positives the heuristic accepts and the dictionary accepts:

- A 15-18 char Id-shaped token that happens to start with a known prefix but is not a real record → caught only when you try to resolve it (404 response)

## Quick reference: where each prefix shows up in flows

| Prefix | Typical context |
|---|---|
| `001` | `recordCreates`/`recordUpdates` of Account |
| `003` | Contact lookups, Opportunity Contact Roles |
| `00D` | Org identity, usually in `<script>` tag of the page (DOM scan only) |
| `0Xl` | Email Consent action input parameters |
| `0eB` | Email Consent action input parameters |
| `0eF` | Email Consent action input parameters |
| `0WW` | CMS-aware flow actions |
| `01p` | Apex action invocations |
| `300` | Subflow references (rare; usually FullName-based) |

## References

- [Salesforce sObject Id field structure](https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce_api_concepts_record_ids.htm)
- [18-character Id case-safe checksum algorithm](https://help.salesforce.com/s/articleView?id=000385717.htm)
- [EntityDefinition.KeyPrefix](https://developer.salesforce.com/docs/atlas.en-us.api_tooling.meta/api_tooling/tooling_api_objects_entitydefinition.htm)
