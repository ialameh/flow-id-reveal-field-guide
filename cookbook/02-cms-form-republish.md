# Cookbook 02: CMS form republish and flow re-patch

When target's CMS form was unpublished, recreated, or has a different content key than expected. Repoint just the form reference.

## Scenario

You have a form-triggered flow in source. The flow's `<form>` reference points at source's CMS content key. You want target's version of the same flow to reference target's CMS form. The flow's Data Cloud Ids are either already correct in target or handled separately.

## Step 1. Inspect the source binding

Side panel → Scan source flow.

Side panel → Inspect → CMS Form → **Show CMS binding**.

Output:

```json
{
  "rawReference": "marketing--Default_Content_Workspace.sfdc_cms__formHandler--MCP4JHBDDOEBHHJKMUYCEKF4YHGY",
  "workspace": "Default_Content_Workspace",
  "contentType": "sfdc_cms__formHandler",
  "contentKey": "MCP4JHBDDOEBHHJKMUYCEKF4YHGY"
}
```

Note the workspace name. Target should have the same workspace name (or you adjust in Step 4).

## Step 2. Find or publish the target form

Side panel → Inspect → CMS Form → **Open target CMS workspace**.

The target CMS workspaces page opens. Navigate to the equivalent workspace and form.

If the form exists:

- Open the form's detail page
- Note the content key (in the URL or detail field)

If the form does not exist:

- Add Content → Form Handler
- Configure fields to match source (or import a definition)
- Publish to the channel
- Note the new content key from the published form's detail page

## Step 3. Switch to target's Flow Builder

Open target's Flow Builder for the equivalent flow. URL: `https://<target>.lightning.force.com/builder_platform_interaction/flowBuilder.app?flowId=<target-flow-draft-id>`.

Make sure you are on the Draft version (the flow you want to patch).

Side panel → Scan → confirm the flow loaded.

## Step 4. Patch with the new content key

Side panel → Inspect → CMS Form → paste the new content key from Step 2 into the input field.

Click **Patch flow with new content key**.

The extension replaces the trailing 20+ char alphanumeric run in the form reference with the pasted key. Confirms before writing.

If the auto-locate fails (because the form reference has unusual format), the dialog prompts to replace the whole reference. Paste the complete new reference in that case.

## Step 5. Verify

Side panel → Inspect → Source → search for the new content key. Confirm it appears in the JSON.

Side panel → Scan → re-scan. The flow metadata should now reference the new key.

## Step 6. Test

Submit the form on the target environment's website (or simulate via Form Submission Event).

Setup → Flows → "Manage" the flow → Total Runs counter increments.

If runs do not increment:

- Verify the form is published in target's CMS
- Verify the channel binding is correct
- Verify the flow is Active in target

## What can go wrong

- Auto-locate fails (form reference has workspace name differences too). Use Search/Replace tab to swap the workspace name first, then re-run CMS Form patch.
- Wrong content key pasted. Re-scan, re-patch with the correct key.
- Target Tooling silent-drop. Use Cross-org deploy with the patched in-memory metadata.

## Time

5-10 min if the target form exists, 15-20 min if you have to publish it.
