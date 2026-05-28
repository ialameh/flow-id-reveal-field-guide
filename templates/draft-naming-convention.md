# Staging Draft naming convention

When you Apply patch to a source Draft (which then contains target-org Ids and is poisoned for source-org runtime), update the Draft's label and description to make its state conspicuous. This prevents accidental activation.

## Label format

```
<original flow name> [STAGING <target>]
```

Examples:

- `General Enquiry AU Form Handler v1 [STAGING UAT]`
- `Lead Routing Flow [STAGING PROD]`
- `Marketing Consent Flow [STAGING DEMO-ORG]`

The bracketed suffix is visible in:

- Flow Builder canvas header
- Setup → Flows list
- Version dropdown

Anyone looking at the flow sees the staging marker before they can Activate.

## Description format

```
*** STAGING DRAFT - DO NOT ACTIVATE ***

This Draft contains <target> org Ids.
Created: <date>
Operator: <name>
Map name: <source-target>
Target deploy mechanism: <Gearset | Change Set | Cross-org deploy>
Restore plan: Patch → Repoint → Prefill INVERSE → Apply
Delete date if not restored: <date + 7 days>

DO NOT activate this Draft in <source> org.
Form submissions will fail because Ids do not resolve here.
```

The description appears when someone clicks "Edit" on the version in Setup, and in some FlowDefinition picker contexts.

## When to apply this convention

- Any time you Apply patch with target Ids
- Any time you have a Draft in source intended for cross-org staging only
- Any time the Draft must not be activated

## When NOT to use this convention

- Drafts being legitimately developed (use normal descriptive labels)
- Active flows
- Obsolete flows (Salesforce manages those statuses automatically)

## Setting label and description

Via Flow Builder:

1. Open the flow
2. Click the flow name in the header to edit the label
3. Setup → Flows → click the flow → click the version → edit Description

Via the extension (v0.6 candidate):

- Auto-rename button in Patch → Repoint that sets the label + description after Apply
- Not yet implemented

## Cleanup convention

After the deploy is verified and source is restored:

- Either restore the original label (remove the bracketed suffix) and clear the description
- Or delete the Draft entirely (Lifecycle → Delete this Draft version)

Stale staging Drafts (week+ old) should be deleted or restored. Long-lived poisoned Drafts are a confused-future-self landmine.
