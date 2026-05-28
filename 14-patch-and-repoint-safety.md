# 14. Patch and Repoint safety

The Patch tab is the only place the extension writes to your orgs. This chapter is the safety model.

## The poisoned-Draft concept

When you click **Apply patch** in the Repoint subtab, the extension:

1. Takes the in-memory `state.flowMetadata` (source's current flow JSON)
2. Replaces every source Id with the corresponding target Id from the Repoint inputs
3. Sends the result back to source org via Tooling API PATCH

After this, the source org's Draft flow version contains target-org Ids. These Ids do not resolve in source. If the source flow ever tries to use them at runtime, every reference throws 404.

The Draft is **poisoned** for source-org runtime use. It is correct for deploy-staging-to-target use.

## What POISONED means in practice

A poisoned Draft cannot be:

- **Activated in source.** Activating it would replace the working Active version with one that breaks every flow run.
- **Tested in source.** A test invocation of the poisoned Draft fails.
- **Forgotten.** The longer it sits, the higher the risk someone unfamiliar with the workflow activates it.

A poisoned Draft can be:

- **Deployed to target.** Gearset / Change Set / sfdx / Extension Cross-org deploy will ship the metadata. Target receives flow with target-correct Ids.
- **Restored.** Prefill INVERSE + Apply puts source Ids back, un-poisoning the Draft.
- **Deleted.** Lifecycle → Delete this Draft version drops it entirely.

## Three operational rules

### Rule 1. Always save the swap to a named map before Apply

In the Repoint subtab, the workflow:

1. Fill target Ids per row
2. Type a map name (`<source>-<target>` is a good convention, e.g. `silverchef-uat`)
3. Click **Save swaps to map**
4. Then Apply patch

The map captures source→target Id pairs. After Gearset deploys, you can prefill INVERSE from the same map to restore source Ids without re-typing.

If you skip the save and Apply, you can still restore by manually reversing the Repoint inputs, but it is friction. The map is one-click.

### Rule 2. Always do a backup before Apply

In the Export tab, **Backup current flow metadata to clipboard** captures the pre-patch state. Save the clipboard contents to a file.

If Apply goes wrong (network error, partial write, unexpected Salesforce behaviour), Restore from clipboard puts the original metadata back via another PATCH.

The Backup is your last-line rollback. Use it.

### Rule 3. Never activate the poisoned Draft in source

The extension's confirm dialog warns about this. The visible safety hint under the Apply button warns. But the click is still possible. Train your team. Document the convention in the Draft's description ("STAGING ONLY - DO NOT ACTIVATE").

If accidentally activated, you have two recovery paths:

- **Restore from Backup** (if you captured one)
- **Re-Activate the previous (good) version** via Setup → Flows → click old version → Activate. The previous Active version's Id is in `state.flowRecord` history if you have it, otherwise look it up in Setup.

Salesforce keeps every version forever. Bad activation = activate the previous good version. Brief window of disruption during the switch.

## The full round-trip pattern

The safe round-trip looks like:

```
[Source org Apply] ←─ Repoint INVERSE ─→ [Source org Restore]
       ↓
[Deploy to target]
       ↓
[Target Activate]
       ↓
[Source Cleanup: Restore OR Delete Draft]
```

Specifically:

1. **Backup** source flow → clipboard
2. **Save** Repoint swaps to map "src-tgt"
3. **Apply** patch → source Draft now has target Ids
4. **Deploy** via Gearset / Change Set / Cross-org / sfdx
5. **Activate** the new Draft in target
6. **Restore** source: Patch → Repoint → Prefill INVERSE from "src-tgt" → Apply
7. Source is back to original

Or step 6 alternative:

6. **Delete Draft** → Lifecycle → Delete this Draft version → source has no staging Draft anymore

The Restore path is preferred when the source Draft has other valuable content beyond the target-Id repointing (e.g. logic improvements you also want to keep). The Delete path is preferred when the Draft was created purely for staging the deploy.

## What if you have to leave source in poisoned state temporarily

Sometimes you Apply, start the Gearset deploy, then need to walk away. The Draft sits poisoned overnight.

Defenses:

- Rename the Draft to make it conspicuous: append " - GEARSET STAGING DO NOT ACTIVATE" to the label. Visible to anyone opening the flow in Setup.
- Set the description to the same. Visible in the FlowDefinition picker.
- Tell the team via Slack / ticket: "Source has a poisoned Draft of <flow>. Do not activate. Will restore tomorrow."

The extension does not currently auto-rename the Draft or update the description. v0.6 candidate.

## Lifecycle button safety

Three buttons in Patch → Lifecycle. Each has confirm dialogs and runs only against the **currently scanned org** + **currently scanned flow record**.

| Button | Effect | Recovery if fired by accident |
|---|---|---|
| **Activate this version** | This version becomes Active. Previously Active becomes Obsolete. | Setup → Flows → click previous version → Activate |
| **Deactivate flow** | No active version. Flow does not fire. | Setup → Flows → click any version → Activate |
| **Delete this Draft version** | Draft permanently removed. | If a backup exists, Restore + recreate as Draft. Otherwise the Draft is gone. |

Always read the confirm dialog. The dialog states which org and which version.

## Cross-org deploy is safer than Apply + Gearset

The Cross-org deploy subtab in Patch lets you ship the in-memory patched metadata directly to target, **without writing to source first**.

The workflow:

1. Scan source → state.flowMetadata loaded
2. Repoint → fill target Ids (but do NOT click Apply)
3. Compare → pick target tab
4. Patch → Cross-org deploy → validate-only first → then real write

Result: source unchanged, target has new Draft with target Ids. No restore step needed. No window of source poisoning.

This is the safest pattern when the deploy tool itself supports it (it does in the extension). Gearset and Change Set are pull-and-push tools, so they need source to already have the patched bytes.

## What if Salesforce silently drops the PATCH

Known Salesforce quirk: Tooling API PATCH on `Flow.Metadata` for form-bound flows sometimes returns 200 OK but does not actually persist the change. The change appears successful in the response but the flow is unchanged when you re-query.

Detection:

- Re-Scan after Apply
- Compare counts before / after
- Check Inspect → Source for the new Id

Fix:

- Use Cross-org MDAPI deploy instead (returns explicit errors, no silent drops)
- Or save the Draft via Flow Builder UI directly (writes through a different code path)
- Or use sfdx retrieve + edit + deploy (authoritative path)

The extension's Audit → Pre-deploy is the workaround. It uses MDAPI deploy validateOnly, which returns real validation errors that surface what Tooling silently dropped.

## References

- [Tooling API Flow object PATCH limitations](https://developer.salesforce.com/docs/atlas.en-us.api_tooling.meta/api_tooling/tooling_api_objects_flow.htm)
- [Flow Builder versioning](https://help.salesforce.com/s/articleView?id=sf.flow_distribute_versioning.htm)
- [Metadata API deploy errors](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_deploy.htm)
