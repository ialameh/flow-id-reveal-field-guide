# 09. Inspect tab

Read-only deep inspection of the active flow. Five subpanels.

## Source

The raw flow metadata JSON.

Click **Load flow source**. The full `state.flowMetadata` blob pretty-prints in the output `<pre>`. Use the search field to find any string with `<mark>` highlighting on matches.

When useful:

- Verifying that a Patch actually wrote the new Ids (search for the target Id prefix)
- Finding the JSON path of an arbitrary value
- Inspecting non-Id structural data (decisions, connectors, loops)
- Copying snippets for ticket descriptions

The source is the same JSON the Export tab's "View raw flow JSON" produces, but displayed in-line with search instead of copy-only.

## Dependencies

Click **Analyze dependencies**. Five sections appear:

| Section | What it shows |
|---|---|
| **Subflows** | Names of subflows called by this flow (from `subflows[].flowName`) |
| **Invocable actions** | Each `actionCalls[]` with its name + type |
| **Apex references** | Subset of actions where the type matches `/apex|apexClass/i` |
| **Custom labels** | Any `$Label.X` reference found anywhere in the metadata |
| **Objects touched** | SObject names appearing in `recordCreates`, `recordLookups`, `recordUpdates`, `recordDeletes` |

Useful for:

- "What does this flow depend on" before refactoring
- Identifying components that need to ship together in a Change Set
- Auditing custom label usage
- Cross-flow impact analysis when renaming a class

## Picklist literals

Click **Extract literals**. Tabulates every `stringValue` literal in the metadata that is NOT a Salesforce Id.

Why this matters: flow decision logic frequently hardcodes picklist values:

```xml
<rules>
  <name>Apply_for_finance</name>
  <conditions>
    <leftValueReference>$Form.nature_of_enquiry</leftValueReference>
    <operator>Contains</operator>
    <rightValue>
      <stringValue>Apply</stringValue>
    </rightValue>
  </conditions>
  ...
</rules>
```

The literal `Apply` is a picklist value. When admin renames it to `Apply for Finance` in the picklist definition, this flow's decision rule still matches `Apply` substring. Behaviour silently drifts.

Use the literals list to:

- Audit which picklist values the flow assumes
- Pre-emptively catch impact of picklist value changes
- Reverse-engineer behaviour when no doc exists

## Field usage

Click **Map field usage**. Outputs an object-keyed JSON map showing what fields the flow reads and writes per sObject.

Reads are from:

- `recordUpdates[].filters[].field` (lookup filter conditions)
- `recordLookups[].filters[].field`
- `recordLookups[].queriedFields`

Writes are from:

- `recordCreates[].inputAssignments[].field`
- `recordUpdates[].inputAssignments[].field`

Output structure:

```json
{
  "Prospect": {
    "writes": ["FirstName", "LastName", "Email", "MobilePhone__c", ...],
    "reads": ["NatureOfEnquiry__c"]
  },
  "Task": {
    "writes": ["Description"],
    "reads": []
  }
}
```

Use this to:

- Confirm which fields target org needs to have (otherwise deploy will fail with "no such field")
- Pre-emptively check FLS profile/permset configuration
- Audit data flow into custom objects

## CMS Form

For form-triggered flows only. The CMS form binding helper.

Click **Show CMS binding**. Parses the `<form>` reference in the Start element into three parts:

```
marketing--Default_Content_Workspace.sfdc_cms__formHandler--MCP4JHBDDOEBHHJKMUYCEKF4YHGY
   workspace                          contentType                 contentKey
```

Output JSON shows the parts separately.

### Deep links

Two buttons:

- **Open source CMS workspace** - opens `<source-origin>/lightning/setup/CMSExperiences/home` in a new tab
- **Open target CMS workspace** - opens `<target-origin>/lightning/setup/CMSExperiences/home` in a new tab (requires target tab selected in Compare → Per-Id check)

Use the workspaces UI to:

- Find the equivalent form in target (if someone published it)
- Publish a new form in target if no equivalent exists
- Grab the new target content key

### Patch with new content key

Once you have the target content key, paste it into the input field and click **Patch flow with new content key (writes Draft)**.

The extension:

1. Replaces the trailing 20+ char alphanumeric run in the form reference with the pasted key
2. Sends `sf-flow-update` with the new metadata
3. Reports the result

If auto-locate of the content-key tail fails (e.g. the form reference uses an unusual format), the extension prompts to replace the WHOLE form reference instead.

See [Chapter 15](15-cms-form-helper.md) for the full CMS form lifecycle.

## What Inspect does not do

- Does not modify flows except via the CMS Form patch button
- Does not validate that referenced fields, subflows, or labels actually exist (run Audit → Pre-deploy for that)
- Does not analyze flow performance (Audit → Perf scan does)
- Does not surface Salesforce-side validation errors (those come from deploy operations)

Pure read except for the CMS Form patch button which writes via Tooling API PATCH.

## References

- [Flow Metadata API field reference](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_flow.htm)
- [Custom label format `$Label.Name`](https://help.salesforce.com/s/articleView?id=sf.cl_about.htm)
- [Salesforce CMS workspace concepts](https://help.salesforce.com/s/articleView?id=sf.cms_workspace.htm)
