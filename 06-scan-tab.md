# 06. Scan tab

The Scan tab is the front door. Surfaces every hardcoded Salesforce Id in the active SF tab, classifies it by sObject prefix, and lets you resolve each Id to its record Name.

## What Scan does

Two scan paths run in parallel and merge.

### Path 1. Tooling API metadata scan (preferred)

If the active SF tab is a Flow Builder page (URL contains `?flowId=301...`), the content script:

1. Extracts the flowId from the URL
2. Sends a `sf-tooling-query` to background
3. Background runs `SELECT Id, Status, VersionNumber, MasterLabel, Metadata FROM Flow WHERE Id='...'`
4. Returns the raw flow Metadata JSON
5. Content script walks the JSON tree, finding every string that contains an Id-shaped token

This is the high-fidelity path. The flow's actual stored metadata is the source of truth.

### Path 2. DOM scan (always runs)

The content script walks the rendered DOM including every open shadow root, plus same-origin iframes. For each element it inspects these attributes:

```
value, data-value, data-record-id, data-recordid, data-id, data-key,
href, data-aura-rendered-by, data-target-selection-name,
data-output-element-id, title, aria-label
```

It also reads direct text node children for Id-shaped strings.

This path is the only option on non-Flow Setup pages. It also catches the org Id (rendered in a `<script>` tag), record links, and the org-internal IDs of Lightning components.

### The merge

Results from both paths are merged by Id. Flow-metadata hits win when there is overlap. The host description shows where the Id was found:

- `actionCalls.0.inputParameters.0.value.apexValue` for flow-metadata hits
- `script[@#text]` for DOM-only hits

## The counts line

After scan, the top of the panel shows:

```
Total 4 | flow metadata: 3 | DOM: 1 | <Flow Label> (301WG00001DY5RJYA1)
```

| Field | Meaning |
|---|---|
| `Total` | Unique Ids after merge |
| `flow metadata` | Ids found via Tooling API scan (0 if not on a Flow Builder page or if Tooling fetch failed) |
| `DOM` | Ids found via DOM scan |
| Flow label + Id | The flow that was scanned (present only when flow metadata path succeeded) |

If `flow metadata: 0` and you are on a Flow Builder page, the Tooling fetch failed. Check the DevTools console of the SF tab for the `[Flow ID Reveal] scanned ...` log line. Common causes: stale session, API version mismatch, network blocked.

## The Resolve buttons

Two ways to resolve.

### Per-row Resolve

Each Id row has its own **Resolve** button. Clicking it calls the org's REST sobject endpoint:

```
GET /services/data/v62.0/sobjects/<SObject>/<Id>
```

The response is parsed and the row's Name column fills with `Name || DeveloperName || MasterLabel || Label`.

### Resolve all found Ids

Bulk version. Fires the same REST calls sequentially (rate-limit polite). Updates the table as each result arrives. For 50+ Ids on a large flow, takes seconds. For 3-10 Ids on a typical flow, near-instant.

## The "Hide unknown prefixes" filter

The classifier knows about 30+ standard sObject prefixes. Ids with unrecognised prefixes are suppressed by default. Uncheck the box to see every Id-shaped token the regex matched, including likely false positives.

When useful:

- You added a Custom Metadata Type record and want to see its Id surface
- A new Salesforce object prefix is not yet in the dictionary
- Debugging why an Id you expect to see is missing (it might be filtered)

To add a prefix to the dictionary, edit both `src/utils/id-patterns.js` and the mirrored constant in `src/content.js`. Commit. Reload the extension.

## Manual resolve subsection

The "Manual resolve" details element at the bottom of Scan accepts pasted Ids (one per line) and resolves them against the active SF tab without scanning. Useful when:

- You have an Id from outside the flow you want to look up
- You are testing the classifier rules on synthetic Ids
- A scan returned 0 hits and you want to verify a specific Id directly

## What gets exported

The Scan tab populates `state.scanned`, which is read by:

- Compare tab (per-Id check uses these Ids)
- Patch → Repoint tab (builds rows from these Ids)
- Export tab → Copy JSON / Copy CSV
- Inspect → Source viewer (uses `state.flowMetadata` captured during scan)

Re-scanning replaces `state.scanned` and `state.flowMetadata`. Run scan whenever you change which SF tab is focused or after a Patch Apply.

## Performance characteristics

Scan times observed on real flows:

- Small flow (1-5 actions, 0 subflows): ~150ms total
- Medium flow (10-30 actions, simple decisions): ~300ms
- Large flow (50+ actions, multiple subflows, big apexValue blobs): ~1-2 seconds

The dominant cost is the Tooling API round-trip (network latency + Salesforce serialization). The local JSON walking is microseconds.

DOM scan on a complex Lightning page with 5k elements + shadow DOM takes ~100-200ms. This is the same regardless of flow complexity.

## What Scan does not do

- Does not modify anything in any org
- Does not deactivate, activate, or delete versions
- Does not call the metadata API or write back
- Does not cache between scans (every scan fetches fresh)
- Does not resolve names automatically (you click Resolve)
- Does not validate that Ids are still live records (only resolves names, surfaces 404 if record is deleted)

Pure read operation. Safe to run on production tabs.

## Common Scan outcomes by flow type

| Flow type | Expected Scan output |
|---|---|
| Record-triggered, no Apex actions, no subscriptions | Just the org Id (DOM). No flow-metadata Ids. |
| Schedule-triggered, calls one Apex class | Org Id + ApexClass Id (`01p...`). |
| Form-triggered Marketing Cloud Growth | 3 Marketing IDs + org Id + maybe CMS content key (not classified by default, see Chapter 15). |
| Subflow that calls multiple subflows | Subflow target FullNames (not Ids in the modern schema). |
| Complex Apex-action flow with hardcoded record lookups | Variable number of standard sObject Ids. |

## References

- [Flow Metadata API reference](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_flow.htm)
- [Tooling API Flow object](https://developer.salesforce.com/docs/atlas.en-us.api_tooling.meta/api_tooling/tooling_api_objects_flow.htm)
- [Shadow DOM and Lightning Web Components](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc.create_components_intro)
