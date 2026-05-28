# 13. Form-bound flow constraints

Form-triggered flows (Marketing Cloud Growth Form Submission flows) have additional constraints beyond regular flows. This chapter covers them, the errors they generate, and how the extension surfaces and works around each one.

## The constraint family

| Constraint | Salesforce rule | Where you hit it |
|---|---|---|
| **One Draft per form binding** | A given CMS form can be bound to at most one Draft flow version at a time. | Trying to create a second Draft when one already exists for the same form. |
| **Published form must have an Active flow** | If the CMS form is published, an Active flow must be bound to it. | Trying to deactivate the only Active version. |
| **Form reference is org-specific** | The `<form>` content key is unique per org. | Cross-org deploy of the metadata as-is. |
| **Form must exist before flow can save** | Flow Builder save fails if the referenced form does not exist in this org. | First-time deploy to a fresh target org. |

Each generates a distinct Salesforce error.

## Error 1. "Form already associated with a draft flow version"

Full text:

```
The form you selected is already associated with a draft flow version.
You can't associate a form to more than one draft flow version.
```

When it fires:

- "Save as new version" in source when a Draft already exists
- Change Set / Gearset / MDAPI deploy of a flow when target already has a Draft for the same form
- Extension's Cross-org deploy when target has an existing Draft

The form, not the flow, is the constrained resource. Multiple flows could theoretically bind to the same form (though Salesforce blocks that too); within one flow, only one Draft per form is allowed.

### Fix

In whichever org has the conflict (often the target during deploy):

- **Activate the existing Draft** → becomes the new Active, frees the form for a new Draft
- **Delete the existing Draft** → frees the form

The extension's **Compare → Draft collision check** detects this before deploy. The extension's **Patch → Lifecycle** subtab has one-click Activate / Delete buttons.

## Error 2. "Activate a flow to publish the form handler"

Full text:

```
To publish the form handler '<form name>', activate its related flow.
```

When it fires:

- Trying to deactivate the only Active version of the bound flow
- The CMS form is currently published

Salesforce enforces: a published form must always have an Active flow to process submissions. Without one, submissions would have no automation, which Salesforce considers invalid state.

### Fix

Two paths:

- **Unpublish the form first** (Setup → CMS Workspaces → form → Unpublish). Now the flow can be deactivated. Re-publish after.
- **Don't deactivate.** Use the extension's Cross-org deploy or another tool that doesn't require deactivation.

The unpublish path is intrusive: submissions during the unpublish window are lost.

## Error 3. Cross-org form GUID mismatch

The form reference in the flow XML:

```xml
<form>marketing--Default_Content_Workspace.sfdc_cms__formHandler--MCP4JHBDDOEBHHJKMUYCEKF4YHGY</form>
```

The trailing GUID is the **content key**, assigned by source CMS when the form was published there. Target CMS has its own content key for any equivalent form (different GUID).

Deploying the flow as-is to target fails because target's CMS does not have a form with that content key. Salesforce returns an error referencing the missing form.

### Fix

The CMS Form helper in the Inspect tab walks you through:

1. Show the parsed form binding
2. Deep-link to source CMS workspace (verify form details)
3. Deep-link to target CMS workspace (publish equivalent form, grab new content key)
4. Paste new content key + Patch the flow's `<form>` reference

See [Chapter 15](15-cms-form-helper.md) for the full helper documentation.

## Error 4. "Form not found"

When it fires:

- First-time deploy of a form-triggered flow to a fresh target org with no equivalent CMS form
- Form was deleted or unpublished in target after the flow was authored

### Fix

Publish the form in target's CMS first. Use the CMS Form helper deep-link to navigate there. Once the form exists with a content key, patch the flow reference, then deploy.

## Why Salesforce enforces these

Form-triggered flows are user-facing automation. A misconfiguration breaks website submissions silently. The constraints exist to prevent:

- Two competing automations for the same form (one Draft conflicts with the Active runtime)
- Submissions disappearing because no flow handles them (published form, no Active flow)
- Cross-org deploys that look successful but the flow can't fire because the form reference is dead

The constraints are annoying during dev but right by design.

## Combined workflow for a form-bound flow

Putting all four together, the safe deploy pattern is:

```
1. Extension → Scan source flow (V_n Active or V_(n+1) Draft)
2. Extension → Compare → identify cross-org Id rot
3. Extension → Compare → Clone or Use this Id per row
4. Extension → Inspect → CMS Form → verify binding parses correctly
5. Manually publish equivalent form in target's CMS, grab new content key
6. Extension → Inspect → CMS Form → paste target content key → Patch
   (Source Draft now has target form GUID baked in)
7. Extension → Patch → Repoint → fill target Ids → Apply
   (Source Draft now has target Data Cloud Ids baked in)
8. Extension → Compare → Draft collision check on target → must be OK
9. Deploy via Gearset / sfdx / Extension Cross-org deploy
10. In target: Extension → Patch → Lifecycle → Activate new Draft
11. In source: Extension → Patch → Repoint → Prefill INVERSE → Apply (restore)
    AND Inspect → CMS Form → patch with source content key → Apply (restore)
    OR Lifecycle → Delete this Draft version (drop the staging Draft)
12. Verify form submissions work end-to-end in target
```

## What if the flow is NOT form-triggered

Most of this chapter does not apply. Regular flows (record-triggered, schedule-triggered, autolaunched) have no form binding, so the form-uniqueness constraint and form-required-for-published-handler rule do not apply.

You still hit the Data Cloud Id rot if the flow uses Marketing Cloud Growth actions (createConsent), but the form chain is absent. Simpler deploys.

## Programmatic detection

To check if the active flow is form-triggered, look at the Metadata blob:

```js
state.flowMetadata.start?.triggerType === "FormSubmissionEvent"
```

The Inspect → CMS Form tab does this check. The Show CMS binding button reports "No form reference" for non-form flows.

## References

- [Marketing Cloud Growth form handlers](https://help.salesforce.com/s/articleView?id=sf.mcg_form_handler_flow.htm)
- [CMS workspace publishing](https://help.salesforce.com/s/articleView?id=sf.cms_workspace_publish.htm)
- [Flow trigger types](https://help.salesforce.com/s/articleView?id=sf.flow_concepts_trigger.htm)
- [Form Submission Event reference](https://help.salesforce.com/s/articleView?id=sf.mcg_form_submission_event.htm)
