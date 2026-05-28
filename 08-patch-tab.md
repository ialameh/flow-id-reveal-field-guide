# 08. Patch tab

Where Ids get rewritten and flows get deployed. Six subpanels.

## Repoint

The primary patch interface.

A table with one row per known Id from the last Scan. Columns: Object, Source Id, Source Name, Target Id (paste input).

### Filling target Ids

Three ways to populate the Target Id column:

1. **Manual paste** - type or paste the target Id into each row
2. **Use this Id** from Compare → Per-Id check on EXISTS rows (autofills)
3. **Clone → target** from Compare → Per-Id check on MISSING rows (autofills with the new cloned Id)
4. **Prefill from saved map** - load Ids from a previously-saved org-pair map

### Apply patch

Once rows are filled, click **Apply patch**.

The extension:

1. Validates the flow record's Status. If not `Draft`, refuses to write. Tooling API PATCH on non-Draft is rejected by Salesforce anyway, but the extension fails early with a clear message.
2. Collects swaps: `{ sourceId: targetId }` for every row with a valid target Id.
3. Stringifies the in-memory `state.flowMetadata`, runs `string.split(from).join(to)` for each swap, parses back.
4. Sends `sf-flow-update` to background with the patched metadata.
5. Background does `PATCH /services/data/v62.0/tooling/sobjects/Flow/<flowId>` with body `{ Metadata: patched }`.
6. Shows the response in the output panel.

If success, `state.flowMetadata` is updated in memory to the patched version. The next Scan should reflect the new Ids.

### The poisoned-Draft warning

Conspicuous warning under the Apply button:

> Safety: patched Draft is POISONED for source-org runtime - target-org Ids inside. Do NOT activate it in source. Use as Gearset deploy-staging only. After Gearset pushes to target, prefill INVERSE + Apply to restore source Ids, or delete the staging Draft entirely.

This is critical. The source Draft after Apply contains target-org Ids that do not resolve in source. If activated in source, every flow run hits 404 on those references and fails.

Treat the patched Draft as a deploy-staging artefact. Either:

- Push it via Gearset / Change Set / Cross-org deploy
- After push lands, restore source Ids (Prefill INVERSE + Apply)
- Or delete the Draft (Lifecycle → Delete)

See [Chapter 14](14-patch-and-repoint-safety.md) for the full safety model.

### Save swaps to map

A persistence layer for Repoint pairs.

Enter a map name (`silverchef-uat`, `prod-to-fullsandbox`, etc.) and click **Save swaps to map**. The current Repoint values are saved into `chrome.storage.local` under that map name.

Future scans of the same flow can:

- **Prefill from saved map** → loads the saved swaps into the Repoint input columns
- **Prefill INVERSE (restore source)** → loads the saved swaps in reverse, ready to restore source Ids after a deploy round-trip

The maps subpanel (described below) lets you view, export, import, or clear all saved maps.

## Search / Replace

Generic find-replace across the flow metadata JSON.

Useful for:

- Replacing a CMS content key (after publishing the form in target - see Chapter 15)
- Repointing references not covered by the prefix dictionary
- One-off swaps that do not warrant a Repoint table

Workflow:

1. Type the "from" string
2. Type the "to" string
3. Click **Preview hits** to see the count
4. Click **Apply search/replace** to PATCH the Draft

Same Draft-only safety rule as Repoint. Same `state.flowMetadata` mutation.

## Cross-org deploy

Direct MDAPI deploy from extension to target. Bypasses Gearset and Change Set.

### Setup

1. Open the target SF tab. Log in.
2. Side panel → Patch → Cross-org deploy → **Refresh target tab list** → click target tab

### Validate-only vs write

Default is **Validate only checked**. The deploy runs as MDAPI `checkOnly=true`. Target is not modified. The deploy validator runs all the same checks it would for a real deploy and returns success or detailed errors.

Always run validate-only first.

Untick the checkbox to switch to a real write (`checkOnly=false`). Target gets a new Draft version of the flow.

### What gets shipped

The extension takes the IN-MEMORY `state.flowMetadata` (which is the source's metadata, with any Repoint swaps applied, even if you have not clicked Apply yet). Generates a `.flow-meta.xml` from the JSON, packages it with a `package.xml`, ZIPs in-memory, base64-encodes, and POSTs to target's MDAPI SOAP endpoint.

The crucial detail: the source-org flow's Active version is not touched. The source-org Draft is not touched. Only the in-memory snapshot, with any pending Repoint swaps, gets shipped.

This means you can:

- Scan source, fill Repoint table with target Ids
- Without clicking Apply on source, click Cross-org deploy
- Source is completely unchanged
- Target receives a new Draft with the right Ids

### The deploy lifecycle

Once submitted, the MDAPI returns an `asyncId`. The extension polls `checkDeployStatus` every 2 seconds for up to 60 seconds. The poll output appears in the output panel, ending with the success or failure verdict + full error detail.

### When Cross-org deploy is the right answer

- You cannot pick a Draft via Change Set (always picks Active)
- Gearset is not set up or licensed
- You want a one-flow surgical deploy without ceremony
- You want fast iteration (Apply patch in source breaks source temporarily; Cross-org deploy never touches source)

See [Chapter 16](16-predeploy-and-zip.md) for the inline ZIP builder and SOAP envelope details.

## Lifecycle

Three utility buttons for managing the flow's activation state.

| Button | Action | Where it runs |
|---|---|---|
| **Activate this version** | PATCH FlowDefinition.activeVersionNumber to current version | Current scanned org |
| **Deactivate flow** | PATCH FlowDefinition.activeVersionNumber to 0 | Current scanned org |
| **Delete this Draft version** | DELETE Flow record | Current scanned org (only works on Draft) |

Common uses:

- After Cross-org deploy lands a new Draft in target, switch tabs to target and click **Activate this version**
- After Gearset push lands and you want to drop the staging Draft in source, click **Delete this Draft version**
- Activate / Deactivate to satisfy Salesforce's "form must have an active flow" rule before any deploy that needs to reshape activation state

Each action shows a confirm dialog. The output panel shows the API response.

## Auto-pin to CMT (v0.4 - suggestion only)

The Custom Metadata Type pattern is the long-term fix for cross-org Id rot. Instead of hardcoding `0XlWG0000000Ihh0AE` in the flow, you create a CMT record per environment and the flow looks up the right Id at runtime.

Click **Suggest CMT extractions**. The extension prints a recommended CMT schema and the records to create in each environment. v0.4 is suggestion-only. v0.5 will add a button to actually create the CMT type + records via Metadata + Tooling API.

See [Chapter 17](17-auto-pin-to-cmt.md) for the full design.

## Org-pair maps

The persistent Id mapping store. Shows the contents of `chrome.storage.local["fir.idMaps"]` as JSON.

Buttons:

- **Reload** - re-read from storage
- **Copy JSON** - copy the map JSON to clipboard
- **Import from clipboard** - paste map JSON to overwrite the store
- **Clear all** - wipe the store

The map structure:

```json
{
  "silverchef-uat": {
    "0XlWG0000000Ihh0AE": "0XlWF0000000GrB0AU",
    "0eBWG00000006rp2AA": "0eBWF00000005HR2AY",
    "0eFWG0000002ABN2A2": "0eFWF000000091h2AA"
  },
  "silverchef-prod": {
    "0XlWG0000000Ihh0AE": "0XlPR0000000XYZ123",
    ...
  }
}
```

Map keys are user-chosen labels for the source-target org pair. Each map is a flat source-Id → target-Id dictionary.

Use cases:

- Build the map over multiple flow Patches (CommSubscription one day, ApexClass next week)
- Export the map for team sharing or backup
- Restore source Ids after Gearset round-trip via Prefill INVERSE

## What Patch does not do

- Does not validate target Ids exist before Apply (Repoint accepts any 15/18-char alphanumeric)
- Does not roll back automatically (you back up via Export → Flow backup, or restore via Repoint INVERSE)
- Does not commit changes anywhere outside the user's browser + the orgs they hit
- Does not handle multi-flow patches in one click (one flow at a time)

## References

- [Tooling API Flow.Metadata PATCH semantics](https://developer.salesforce.com/docs/atlas.en-us.api_tooling.meta/api_tooling/tooling_api_objects_flow.htm)
- [Metadata API deploy](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_deploy.htm)
- [FlowDefinition activation](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_flowdefinition.htm)
- [chrome.storage API](https://developer.chrome.com/docs/extensions/reference/api/storage)
