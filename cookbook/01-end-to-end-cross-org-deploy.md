# Cookbook 01: End-to-end cross-org deploy

The complete dance. Source flow with hardcoded Ids → target org with valid Ids and working form trigger. 30 minutes.

## Prerequisites

- Source org with a form-triggered flow (Marketing Cloud Growth Form Submission flow recommended)
- Target sandbox or org you have write access to
- Both logged in via Chrome
- Flow ID Reveal extension installed and pinned

## Step 1. Backup the source flow

Side panel → Export → Flow backup / restore → **Backup current flow metadata to clipboard**.

Save the clipboard contents to a file: `~/Desktop/flow-backup-<flow-name>-pre-patch.json`. This is your rollback artifact.

## Step 2. Scan source

Side panel → Scan → **Scan active SF tab**.

Expect output similar to:

```
Total 4 | flow metadata: 3 | DOM: 1 | <flow label> (301WG00001DY5RJYA1)
```

Click **Resolve all found Ids**. The Name column fills.

## Step 3. Switch to target tab and resolve Ids there

Open the target SF tab in the same Chrome window. Log in.

Switch back to the side panel. Side panel → Compare → **Refresh tab list**. Click the target tab row.

Click **Compare against selected target**.

For each row, the result is EXISTS or MISSING:

- **EXISTS** → click **Use this Id** in the action column to autofill the Repoint table
- **MISSING** → click **Clone → target** to auto-create the equivalent record + autofill

## Step 4. Handle CMS form (if form-triggered)

Side panel → Inspect → **CMS Form** → click **Show CMS binding**.

If the binding shows a form reference:

1. Click **Open target CMS workspace**. The CMS workspaces page opens in a new tab.
2. Find or publish the equivalent form in target's CMS. Note the new content key.
3. Back in side panel, paste the new content key into the input field
4. Click **Patch flow with new content key**

Source Draft now references the target's form content key. Verify via Inspect → Source → search for the new key.

## Step 5. Patch source Draft with target Ids

Side panel → Patch → Repoint.

The table should be populated from Step 3 (Use this Id / Clone → target).

Type a map name: `source-target` (or similar). Click **Save swaps to map**.

Click **Apply patch**.

The poisoned-Draft warning appears in the confirm dialog. Read it. Confirm.

Result: source Draft has target Ids baked in. Source is now in staging state. Do not activate it.

## Step 6. Check target for Draft collision

Side panel → Compare → **Draft collision check** subtab → click **Check target for Draft collisions**.

Verdict:

- **OK** → proceed to Step 7
- **BLOCKED** → switch to target tab, scan target's flow Draft, side panel → Patch → Lifecycle → Activate or Delete. Then re-run collision check.

## Step 7. Pre-deploy validate

Side panel → Audit → **Pre-deploy** → click **Run pre-deploy validation**.

The deploy validator runs in target org. Wait for the verdict.

- **PASSED** → safe to deploy. Proceed to Step 8.
- **FAILED** → read the error. Common causes:
  - Form not found (Step 4 incomplete)
  - Custom field missing in target (deploy field first)
  - Form-binding collision (Step 6 incomplete)

Fix the issue, re-run validate, then proceed.

## Step 8. Deploy via Cross-org deploy (or Gearset)

### Option A: Cross-org deploy

Side panel → Patch → **Cross-org deploy** → click **Refresh target tab list** → tick target tab.

**Untick** "Validate only" (we already validated in Step 7).

Click **Deploy patched metadata to selected target**.

Wait for the verdict in the output panel. Target now has a new Draft with target Ids.

### Option B: Gearset

Run your Gearset deploy. Source has the patched Draft. Gearset retrieves it and pushes to target.

Either way, target should now have a new Draft of the flow with valid target Ids.

## Step 9. Activate the new Draft in target

Switch to target tab. Side panel → Scan → confirm the new Draft loaded.

Side panel → Patch → Lifecycle → **Activate this version**.

Target's flow is now Active with target-correct Ids.

## Step 10. Verify end-to-end

Submit the form on target's website (or use Form Submission Event simulator in Setup).

Setup → Flows → "Manage" the flow → check Total Runs counter increments.

If runs do not increment, see [Chapter 13](13-form-bound-flow-constraints.md) and [Chapter 15](15-cms-form-helper.md) for form-binding and CMS troubleshooting.

## Step 11. Restore source

Switch back to source tab. Side panel → Patch → Repoint → enter the map name from Step 5 → click **Prefill INVERSE (restore source)**.

The Repoint table fills with target → source swaps.

Click **Apply patch**.

Source Draft is back to source Ids. Safe to keep, safe to activate if you want.

OR, if you don't want the staging Draft at all:

Side panel → Patch → Lifecycle → **Delete this Draft version**.

The Draft is gone. Source Active version is untouched.

## Step 12. Update CMS form reference in source

If you patched the CMS form reference in Step 4, restore it too:

Side panel → Inspect → CMS Form → paste source content key → Patch.

Or use Patch → Search/Replace to swap target's content key for source's content key.

This step is needed only if you kept the source Draft (instead of deleting it in Step 11).

## What you have now

- Target org: flow Active with correct target Ids, CMS form bound correctly, form submissions work
- Source org: original state restored or staging Draft deleted, source flow continues to work normally

## Time breakdown (typical)

| Step | Time |
|---|---|
| Backup, scan source, compare | 3 min |
| Clone any missing records | 2 min per record (variable) |
| CMS form publish in target | 5-10 min (manual UI work) |
| Apply patch, collision check, validate | 2 min |
| Deploy (Cross-org or Gearset) | 1-3 min |
| Activate in target | 1 min |
| Restore source | 1 min |
| Verify end-to-end | 5 min |
| **Total** | **20-30 min** |

## What can go wrong

- Step 3: Clone fails due to required-field-not-in-source. Add the field manually via UI, retry.
- Step 5: Apply succeeds but flow unchanged (Tooling silent-drop). Use Cross-org deploy instead.
- Step 7: Validate fails on a non-Id issue (custom field missing). Pause this workflow, deploy the missing component, resume.
- Step 8: Deploy fails on form-binding (Step 6 was stale). Re-check, fix, re-deploy.
- Step 9: Activation fails ("invalid Draft"). Some required validation failed. Open the flow in Flow Builder and inspect for errors.
