# 15. CMS Form helper deep dive

CMS forms add a layer of cross-org Id rot beyond Data Cloud record Ids. The CMS Form helper in the Inspect tab handles this layer.

## Why CMS is in the chain

A Marketing Cloud Growth Form Submission flow has four components in series:

```
[CMS Workspace] → [CMS Form content item] → [Channel publish] → [Flow trigger binding]
```

The flow's Start element references the form by its **content key**, not its sObject Id (because CMS content items are not sObjects, see Chapter 05 for the Id taxonomy distinction).

When the flow ships to another org, the form must exist there with its own content key, and the flow's reference must point at the target's content key.

## Anatomy of the form reference

```
marketing--Default_Content_Workspace.sfdc_cms__formHandler--MCP4JHBDDOEBHHJKMUYCEKF4YHGY
```

Parsed:

| Part | Value | Meaning |
|---|---|---|
| Namespace prefix | `marketing--` | Salesforce managed namespace for Marketing Cloud Growth |
| Workspace name | `Default_Content_Workspace` | The CMS Workspace developer name |
| Content type | `sfdc_cms__formHandler` | CMS content type identifying this as a form-handler |
| Content key prefix | `--` | Separator |
| Content key | `MCP4JHBDDOEBHHJKMUYCEKF4YHGY` | The form's unique GUID in this org |

The workspace name and content type are usually consistent across orgs (if you use the same naming convention). The content key is always different.

## The helper UI

Three sections.

### Show CMS binding

Click. Parses the form reference (or shows "No form reference" if the flow is not form-triggered) into the JSON parts. Use this to verify the binding before patching.

### Deep-link buttons

- **Open source CMS workspace** opens `<source-origin>/lightning/setup/CMSExperiences/home` in a new browser tab
- **Open target CMS workspace** opens the target org's CMS Workspaces page (requires target tab selected in Compare → Per-Id check)

From there you:

- Navigate to the workspace (`Default Content Workspace` or wherever your forms live)
- Find or publish the equivalent form
- Open the form's detail page
- Read the URL or detail page for the content key

### Paste + Patch

Once you have the target content key, paste it into the input field. Click **Patch flow with new content key (writes Draft)**.

The extension:

1. Replaces the trailing 20+ char alphanumeric run in the form reference with the pasted key
2. Sends `sf-flow-update` to background with the new metadata
3. Tooling API PATCH writes the change to the currently scanned Draft

If the auto-locate regex fails (form reference uses an unusual format), the extension prompts to replace the WHOLE form reference with the pasted value. Use this when you have the full new reference string, not just the key tail.

## Where to find the content key in target's CMS

Path one:

1. Setup → quick find "CMS Workspaces" → click into the workspace
2. Find the form in the content list
3. Click into the form
4. URL contains the content key, or click "Copy ID" if available
5. Some Salesforce versions show the content key in a detail field

Path two (programmatic):

```sql
SELECT Id, Title, ContentKey, ContentType FROM ManagedContentVersion WHERE ContentType = 'sfdc_cms__formHandler'
```

The `ContentKey` field is what the flow uses. Note: this is a Tooling/Connect API query, not standard SOQL. Use Workbench REST Explorer or the extension's Inspect → Source if you happen to be on a flow tab.

Path three:

Inspect the form's record URL after navigating in target's CMS UI. The 28-char alphanumeric segment in the URL is the content key.

## Publishing a new form in target

If target has no equivalent form:

1. Setup → CMS Workspaces → workspace
2. Add Content → Form Handler
3. Configure fields matching source (or import via CMS API if you have a definition file)
4. Publish to the channel that exposes the form on your website
5. Grab the new content key
6. Paste into the extension's Patch field

This is manual work. Salesforce does not expose form CRUD via REST sobject API. The Connect API supports it but with limitations.

## What if the workspace name differs

If source uses `Default_Content_Workspace` and target uses `Production_Marketing`, the workspace prefix in the form reference also differs:

```
SOURCE: marketing--Default_Content_Workspace.sfdc_cms__formHandler--<source-key>
TARGET: marketing--Production_Marketing.sfdc_cms__formHandler--<target-key>
```

The extension's auto-patch replaces only the content key tail. For a workspace name change, use the prompt to replace the whole form reference.

Or use the Patch → Search/Replace subtab to swap `Default_Content_Workspace` for `Production_Marketing` first, then run CMS Form helper for the content key.

## Cross-org deploy implication

The CMS form GUID must be patched **before** Cross-org deploy. Otherwise target receives a flow referencing a content key that does not exist in target's CMS, and the deploy fails with "form not found".

Workflow ordering:

1. Publish form in target's CMS, get new key
2. CMS Form helper → Patch with new key → source Draft now references target's content key
3. Repoint subtab → fill Data Cloud target Ids → Apply
4. Cross-org deploy or Gearset push

If you skip step 2, the deploy fails on the form reference. Validate-only catches this.

## What if multiple forms map to one flow

Rare. Salesforce supports binding one flow to multiple forms via separate Start branches, but most Form Submission flows bind to exactly one form. The extension's helper only handles the single-form case currently. For multi-form flows, use Search/Replace to swap each form GUID one at a time.

## Testing the new binding

After patching, the flow in target should fire on form submissions. To test:

1. Activate the flow in target (Lifecycle → Activate)
2. Submit the form on the target environment's website
3. Check Setup → Flows → "Manage" the flow → Total Runs counter increments

If runs do not increment, common causes:

- Form is not published in target's CMS (publish it)
- Channel binding missing (the form is published but not exposed on the website)
- Flow is not Active in target (activate it)
- Form fields differ from what the flow expects (e.g. field renamed)

## References

- [Salesforce CMS overview](https://help.salesforce.com/s/articleView?id=sf.cms_workspace.htm)
- [CMS Form Handler documentation](https://help.salesforce.com/s/articleView?id=sf.cms_form_handler.htm)
- [Connect CMS API](https://developer.salesforce.com/docs/atlas.en-us.connectApi.meta/connectApi/connect_resources_cms.htm)
- [ManagedContentVersion sObject](https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce_api_objects_managedcontentversion.htm)
