# Cookbook 03: Bulk Id mapping and restore

You've been doing manual Repoint per flow. Time to build a reusable Id map and use the inverse-restore pattern. 15 minutes for the first flow, < 5 min per subsequent flow.

## Scenario

You have multiple flows in source that all reference the same set of Marketing Cloud Growth records (Marketing CommSub, Email channel, etc.). Each flow needs the same Ids repointed for cross-org deploy.

Building one named map and reusing it across flows saves typing and reduces typo risk.

## Step 1. First flow — build the map

Scan the first flow. Compare against target. Fill the Repoint table via Use this Id / Clone → target as usual.

Enter a map name: `silverchef-to-uat` (or whatever describes your source-target pair). Click **Save swaps to map**.

The map is persisted in `chrome.storage.local`. Visible at Patch → Org-pair maps → click **Reload** → Copy JSON to verify:

```json
{
  "silverchef-to-uat": {
    "0XlWG0000000Ihh0AE": "0XlWF0000000GrB0AU",
    "0eBWG00000006rp2AA": "0eBWF00000005HR2AY",
    "0eFWG0000002ABN2A2": "0eFWF000000091h2AA"
  }
}
```

Click Apply patch on the first flow. Deploy. Activate in target. (Steps from Cookbook 01.)

## Step 2. Second flow — use the saved map

Scan the second flow (a different flow that references the same Marketing Cloud Growth Ids).

Side panel → Patch → Repoint → enter `silverchef-to-uat` in the map name field → click **Prefill from saved map**.

The Repoint table autofills with target Ids for every source Id in the map. Rows that don't match the saved Ids stay blank — fill those manually or skip if not applicable to this flow.

Apply patch. Deploy. Activate in target.

Total time on this second flow: under 5 minutes (most of it is human verification + the Apply confirm dialog).

## Step 3. Third, fourth flow — same pattern

Each subsequent flow that uses the same Id set gets the prefill treatment. The map saves you from re-cloning records and re-typing pasted Ids.

## Step 4. Restore source after deploy round-trip

When you're done deploying for the day and want to restore source flows to their original state:

For each flow that was Patched:

1. Scan the flow in source
2. Side panel → Patch → Repoint → enter `silverchef-to-uat` map name → click **Prefill INVERSE (restore source)**
3. The Repoint table now has target Ids on left, source Ids on right (inverted from before)
4. Click **Apply patch**

Source flow's Draft is back to source Ids. Safe.

## Step 5. Map maintenance

As you discover more Id pairs over time, add them to the map:

- After Compare → Per-Id check, click Save swaps to map again (without changing the name) → new pairs append to the existing map
- After Clone → target produces a new Id, click Save swaps to map to capture the new pair

The map grows over time. When it covers all the cross-org records your org uses, Repoint becomes a one-click operation.

## Step 6. Sharing with team

Side panel → Patch → Org-pair maps → click **Copy JSON**. Paste into:

- A shared Confluence / Notion / Google Doc page
- A git config repo
- A team Slack channel pinned message

Teammates: Side panel → Patch → Org-pair maps → click **Import from clipboard** → paste.

Now everyone on the team has the same Id mappings. Workflow becomes truly reproducible.

## Map naming convention

Suggested patterns:

- `<source-alias>-to-<target-alias>` — `silverchef-to-uat`, `prod-to-fullsandbox`
- `<deploy-cycle>-<sprint>` — `q1-2026-cycle3`
- `<purpose>` — `general-enquiry-bindings`, `marketing-channel-mappings`

Pick a convention and stick to it. Saved maps are flat key-value, no folder hierarchy.

## When to start a new map vs reuse

Start a new map when:

- The source-target pair changes (different sandboxes)
- The Ids drift (source org records were deleted + recreated)
- The use case is fundamentally different

Reuse when:

- Same source + same target
- Same record set (Marketing Cloud Growth Ids stay stable for the lifetime of the records)

## Common workflows

- **Weekly deploy:** keep a `<src>-to-<target>` map per pipeline. Restore source weekly via INVERSE prefill.
- **One-off migration:** build a `migration-<date>` map, use once, archive (export to git).
- **Ongoing maintenance:** master map kept in version control, imported per session, augmented as needed.

## Time savings

| Flows processed | Without maps | With maps |
|---|---|---|
| 1 | 30 min | 30 min |
| 2 | 60 min | 35 min |
| 5 | 150 min | 50 min |
| 10 | 300 min | 75 min |

The first flow has the same cost. Each subsequent flow drops dramatically because the Compare + Clone step is replaced by Prefill from saved map.
