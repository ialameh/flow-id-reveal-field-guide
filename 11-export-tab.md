# 11. Export tab

Data export + flow metadata backup. Two subpanels.

## Scan results

Three buttons.

### Copy JSON

Copies the `state.scanned` array to clipboard as pretty-printed JSON. Format:

```json
[
  {
    "id": "0XlWG0000000Ihh0AE",
    "prefix": "0Xl",
    "length": 18,
    "object": "CommSubscription",
    "known": true,
    "name": "Marketing",
    "hosts": [
      {
        "tag": "json",
        "attr": "actionCalls.0.inputParameters.0.value.apexValue",
        "label": "...",
        "componentName": "actionCalls.0"
      }
    ]
  },
  ...
]
```

Useful for:

- Paste into a ticket describing the cross-org rot
- Save before a major Patch as a manual rollback reference
- Diff against a future scan to detect drift

### Copy CSV

Same data, flattened to CSV columns: `id, object, name, prefix, hosts`. Hosts are joined with `|` separator. Multi-line values are quoted.

Useful for:

- Paste into a spreadsheet for team review
- Feed into a deploy checklist generator
- Audit trail in change management systems that expect tabular evidence

### View raw flow JSON

Bypasses the scan results. Re-fetches the active flow's full metadata via the active tab's content script and displays + copies the pretty-printed JSON.

Useful for:

- Inspecting unscanned fields (decisions, connectors, screen elements, etc.)
- Manual editing prep (copy, edit in your editor, paste back via Search/Replace if needed)
- Backup before a write operation

Also writes to `state.flowMetadata`, so subsequent Inspect → Source loads the same data.

## Flow backup / restore

Manual safety net.

### Backup current flow metadata to clipboard

Captures the current `state.flowMetadata` with metadata about where it came from:

```json
{
  "backedUpAt": "2026-05-28T03:14:59.000Z",
  "flowRecord": {
    "Id": "301WG00001DXvoVYAT",
    "MasterLabel": "General Enquiry - AU Form Handler v1 Flow",
    "VersionNumber": 8,
    "Status": "Draft"
  },
  "origin": "https://silverchefrentalspty--sfomnidev.sandbox.lightning.force.com",
  "metadata": { /* full flow JSON */ }
}
```

Copies to clipboard and shows in output panel.

Run this:

- Before any Apply patch
- Before any Cross-org deploy
- Before any Lifecycle action (Activate / Delete)
- Routinely as a checkpoint when experimenting

Save the JSON somewhere durable (file, gist, Notion). The clipboard is not a backup tool by itself.

### Restore from clipboard

Prompts for backup JSON paste. Validates structure. Confirms before writing.

Writes the backup's `metadata` to the **currently scanned** flow via Tooling PATCH. If you backed up flowId X but are now scanning flowId Y, you can restore X's content into Y (the dialog tells you what is happening).

When useful:

- A Patch went wrong and you have a backup
- You want to copy one flow's structure into another
- You are testing the extension's behavior with known-good metadata

The same Draft-only safety rule applies: Tooling PATCH cannot write to Active versions.

## What gets persisted vs ephemeral

| Data | Lifetime |
|---|---|
| `state.scanned` | Session (refresh side panel = lost) |
| `state.flowMetadata` | Session |
| Org-pair maps | Persistent via `chrome.storage.local` (survives Chrome restart) |
| API version override | Persistent via `chrome.storage.local` |
| Detected API versions per host | In-memory only (service worker lifetime) |
| Backups | Clipboard only — copy elsewhere to persist |

The extension is intentionally light on persistent state. Salesforce's REST endpoints are the source of truth. The extension is a UI on top, not a database.

## Export tab as a CI hook

You can use the Export tab outputs as input to:

- A team-wide deploy planning spreadsheet
- A CI/CD pipeline that pre-validates flows before scheduled deploys
- An incident postmortem describing exactly which Ids broke

Not part of the extension itself, but the JSON / CSV shapes are stable and scriptable.

## What Export does not do

- Does not back up flow versions other than the one currently scanned
- Does not back up subflows, custom labels, or other dependencies (those need their own retrieve)
- Does not version-control backups (Git, S3, etc.)
- Does not encrypt backups (the clipboard is plaintext)

## References

- [navigator.clipboard API](https://developer.mozilla.org/en-US/docs/Web/API/Clipboard)
- [chrome.storage.local persistence](https://developer.chrome.com/docs/extensions/reference/api/storage)
- [Tooling API Flow.Metadata PATCH](https://developer.salesforce.com/docs/atlas.en-us.api_tooling.meta/api_tooling/tooling_api_objects_flow.htm)
