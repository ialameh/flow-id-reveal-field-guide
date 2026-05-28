# 07. Compare tab

Cross-org comparison: the answer to "will my flow deploy to target". Four subpanels.

## Per-Id check

The main workflow.

### Setup

1. Open a second Salesforce tab in the same Chrome window (the target org). Log in.
2. Side panel → Compare → Per-Id check → **Refresh tab list**
3. The tab list shows every SF tab Chrome currently has open
4. Click the row for the target tab. Border turns blue.

### Run the compare

Click **Compare against selected target**.

The extension takes the Ids from the most recent Scan (the `state.scanned` array) and resolves each one against the target tab. The target tab's content script performs the REST sobject GET using target's session cookie.

For each Id, the result is one of:

| Status | Meaning | Action column |
|---|---|---|
| **EXISTS** | Target org has a record with this exact Id | Open source · Open target · **Use this Id** |
| **MISSING (404)** | Target org has no record with this Id | Open source · Create manually · **Clone → target** |
| **ERROR** | Other failure (permissions, API version, etc.) | Open source only |

### Use this Id

When EXISTS, the target Id is by definition the same as source. Sandbox refreshed from same prod within the last refresh interval. Click **Use this Id** to autofill the source-row in the Repoint table with target's Id (which is source's Id in this case, so the Repoint is a no-op for this row).

In practice EXISTS happens on sibling sandboxes refreshed from same prod, or on standard records that prod assigns deterministically.

### Clone → target

When MISSING, the extension can attempt to auto-create the equivalent record in target.

Click. Extension:

1. Fetches the source record (`GET /services/data/v62.0/sobjects/<Type>/<sourceId>`)
2. Strips system fields (`Id`, `OwnerId`, `CreatedDate`, audit fields, `attributes`, null fields, relationship objects)
3. Describes target's sobject to get the createable-field list
4. Filters source fields to those createable in target
5. POSTs the filtered payload to target (`POST /services/data/v62.0/sobjects/<Type>/`)
6. Receives target's new record Id
7. Autofills the Repoint table row for the source Id with the new target Id

A confirm dialog states each step before execution. If POST fails, the error returns inline.

Caveats:

- **Dependency order matters.** Clone parents before children. CommSubscription before CommSubscriptionChannelType.
- **OwnerId is dropped.** Target assigns OwnerId to the user whose session sent the request (you).
- **Some objects reject POST.** System-managed objects (Profile, PermissionSet, ApexClass) are uncreateable via REST. The describe filter detects this and the POST returns 400. Use Create Manually link instead.
- **Required fields the source did not populate.** Salesforce returns a specific error naming the missing field. Fill it (you may need to source it manually) and retry.

### Create manually link

For MISSING rows where Clone is not possible, the **Create manually** link deep-links to target's `/lightning/o/<SObject>/new` page. You author the record in target's UI, copy the new Id, paste it into the Repoint row.

## Cross-org JSON diff

The full-flow comparison.

Pulls source's flow metadata (from `state.flowMetadata`, captured during last scan) and target's flow metadata (via Tooling API from the target tab). Runs a line-based diff on pretty-printed JSON.

Output is HTML-rendered with three colour classes:

- `+ green` lines: present in target, not in source
- `- red` lines: present in source, not in target
- ` grey` lines: unchanged context

The diff is capped at 5000 lines per side to keep memory sane. Larger flows truncate with a notice.

Useful for:

- Spotting which fields differ besides the Ids you already know about
- Verifying that target has the same flow structure (decision logic, action wiring)
- Confirming that no element drifted between the orgs

## Draft collision check

A safety check before deploying a form-triggered flow.

Salesforce enforces: one Draft per Form binding per org. If target already has a Draft for the same flow name, your deploy will fail with:

```
The form you selected is already associated with a draft flow version.
You can't associate a form to more than one draft flow version.
```

The check runs a Tooling SOQL:

```sql
SELECT Id, MasterLabel, VersionNumber, Status, LastModifiedDate
FROM Flow
WHERE MasterLabel='<current flow label>' AND Status='Draft'
```

against the target tab's org. Returns one of two verdicts:

- **OK - no Draft collision. Deploy should land.**
- **BLOCKED - N Draft(s) bound to this flow's name in target. Activate or delete before pushing.**

If BLOCKED, the next step is the Lifecycle subtab (Patch tab) in the target org's context. Activate the existing Draft, or Delete it, then re-run the check.

## Version diff

For comparing two versions of the SAME flow in the SAME org. Different from the cross-org JSON diff.

Click **Load versions**. Extension runs:

```sql
SELECT Id, VersionNumber, Status, LastModifiedDate
FROM Flow
WHERE DefinitionId = '<this flow's definition>'
ORDER BY VersionNumber DESC
```

Displays a table with checkboxes in two columns: A and B. Tick one A and one B. Click **Diff selected versions**.

Runs the same line-based diff as cross-org JSON diff but against the same-org Tooling API for the two version Ids. Surfaces what changed between V_n and V_(n+1).

Useful for:

- "What did the last edit actually change"
- Verifying a Draft hasn't drifted from its parent Active version in unexpected ways
- Audit trail when no commit history exists

## What Compare does not do

- Does not deploy or modify either org (read-only across Compare tab except for Clone → target)
- Does not handle field-level permissions (just record existence)
- Does not check field-by-field equality between source and target (only Id resolution)
- Does not validate that an EXISTS Id is the "right" record (it might be a different record sharing the Id, which is technically impossible on the same pod but possible if records were deleted and re-created at the same prefix sequence position)

## Tab persistence

The selected target tab persists in `state.selectedTargetTabId` for the session. It is shared across:

- Per-Id check
- Cross-org JSON diff
- (Optionally) Cross-org deploy on the Patch tab

If you change which target tab you want to compare against, re-click the tab in the list. Selection is single-tab.

## References

- [Tooling API Flow object queries](https://developer.salesforce.com/docs/atlas.en-us.api_tooling.meta/api_tooling/tooling_api_objects_flow.htm)
- [sObject describe endpoint](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/dome_describe.htm)
- [Form Submission Event documentation](https://help.salesforce.com/s/articleView?id=sf.mcg_form_submission_event.htm)
