# 17. Auto-pin to Custom Metadata

The long-term fix for cross-org Id rot. Instead of hardcoding Ids in the flow, store them in Custom Metadata Type records and look them up at flow runtime.

## The pattern

Without CMT (the today situation):

```
Flow XML:
  apexValue: "{...commSubscriptionId: '0XlWG0000000Ihh0AE'...}"
```

Hard-baked. Org-specific. Breaks on cross-org deploy.

With CMT:

```
Flow:
  Get Records → Flow_Environment_Bindings__mdt where DeveloperName = 'Marketing_CommSubscription'
  Assignment → varCommSubId = <retrieved record>.Record_Id__c
  ...later... apexValue: "{...commSubscriptionId: {!varCommSubId}...}"

Org-specific CMT records:
  Source: Flow_Environment_Bindings__mdt.Marketing_CommSubscription.Record_Id__c = '0XlWG0000000Ihh0AE'
  Target: Flow_Environment_Bindings__mdt.Marketing_CommSubscription.Record_Id__c = '0XlWF0000000GrB0AU'
```

Flow XML is identical across orgs. The variation lives in the CMT records, which are environment-specific by design and deployed separately.

Deploy of the flow no longer hits Id rot because the flow does not contain any record Ids. CMT records still need to be deployed, but they are smaller, simpler, and intentionally environment-aware.

## Why this is hard to retrofit

Existing flows have hardcoded Ids in many places: action inputAssignments, decision conditions, JSON apexValue blobs, screen component data references, etc. Each one needs:

1. A new flow variable
2. A Get Records element before the consumer
3. The reference rewritten to use the variable

For a flow with 5 hardcoded Ids, this can mean 5 new Get Records + 5 new variables + 5 reference rewrites. The flow becomes longer but more portable.

The extension's Auto-pin to CMT subpanel (currently v0.4 suggestion-only) prints the plan: which CMT records to create + which flow edits to make. v0.5 will automate creation of the CMT records via Tooling API. Flow refactoring is harder to automate and stays manual.

## Recommended CMT schema

For Marketing Cloud Growth + Data Cloud bindings:

```
Custom Metadata Type: Flow_Environment_Bindings__mdt
Fields:
  Label             Text(80)        Friendly name (e.g. "Marketing Email Subscription")
  DeveloperName     Text(40)        API-safe name (e.g. "Marketing_CommSubscription")
  Record_Id__c      Text(18)        The Id (env-specific)
  Object_Type__c    Picklist(...)   CommSubscription | CommSubscriptionChannelType | etc.
  Environment__c    Picklist        Dev | UAT | SilverChef | Prod (informational only)
  Notes__c          Long Text       Free-form context
```

Each CMT record represents one env-specific binding. The flow looks up by `DeveloperName` (stable across envs) and reads `Record_Id__c` (env-specific).

For CMS form GUIDs and CMT picklist literals (Chapter 09 picklist extractor), use a separate CMT type:

```
Custom Metadata Type: Flow_Form_Bindings__mdt
Fields:
  Label
  DeveloperName
  Workspace__c     Text(80)
  Content_Type__c  Text(80)
  Content_Key__c   Text(128)
```

Same lookup pattern, different field shape.

## The flow refactor

For each hardcoded Id at a flow position, e.g. the `engagementChannelTypeId` in the `Email_Consent` action:

### Before

```xml
<actionCalls>
  <name>Email_Consent</name>
  <actionType>createConsent</actionType>
  <inputParameters>
    <name>consentRequest</name>
    <value>
      <apexValue>{"recordList":[{"engagementChannelTypeId":"0eFWG0000002ABN2A2",...}]}</apexValue>
    </value>
  </inputParameters>
</actionCalls>
```

### After

```xml
<recordLookups>
  <name>Get_Email_Channel_Binding</name>
  <object>Flow_Environment_Bindings__mdt</object>
  <filters>
    <field>DeveloperName</field>
    <operator>EqualTo</operator>
    <value><stringValue>Email_Engagement_Channel</stringValue></value>
  </filters>
  <queriedFields>Record_Id__c</queriedFields>
  <connector><targetReference>Email_Consent</targetReference></connector>
</recordLookups>

<actionCalls>
  <name>Email_Consent</name>
  <actionType>createConsent</actionType>
  <inputParameters>
    <name>consentRequest</name>
    <value>
      <!-- Now references a dynamic value built from the CMT lookup -->
      <apexValue>{"recordList":[{"engagementChannelTypeId":"{!Get_Email_Channel_Binding.Record_Id__c}",...}]}</apexValue>
    </value>
  </inputParameters>
</actionCalls>
```

The apexValue contains a flow-time merge field `{!Get_Email_Channel_Binding.Record_Id__c}` which Salesforce substitutes at execution time with the value from the CMT record matching the lookup.

## Deploy mechanics

CMT type + records ship via Metadata API. From sfdx:

```bash
sf project deploy start --metadata CustomMetadata
```

Or via Change Set (CMT records are deployable via Change Set).

The flow ships separately (Metadata API or Change Set). Because the flow no longer contains environment-specific Ids, it deploys to any target without Id rot.

Per-environment record values can be different:

- Dev environment: `Marketing_CommSubscription.Record_Id__c = '0XlDV000...'`
- UAT environment: `Marketing_CommSubscription.Record_Id__c = '0XlUT000...'`
- Prod environment: `Marketing_CommSubscription.Record_Id__c = '0XlPR000...'`

Deploy strategy options:

1. **One CMT record per environment, all in source control.** Ship all records; each env reads its own DeveloperName-keyed record.
2. **CMT records as data per env, deployed at env setup time.** Not source-controlled. Less clean but works.
3. **Hybrid: CMT type in source control, records bootstrapped per environment.** Most production-grade pattern.

## Extension's suggestion output

v0.4 prints (when you click Suggest CMT extractions):

```json
{
  "proposedCmt": "Flow_Environment_Bindings__mdt",
  "fields": [...],
  "suggestedRecords": [
    {
      "DeveloperName": "CommSubscription_Marketing",
      "Label": "CommSubscription: Marketing",
      "Record_Id__c": "0XlWG0000000Ihh0AE",
      "Environment__c": "<source>"
    },
    ... one per hardcoded Id ...
  ],
  "suggestedFlowEdits": [
    "Replace literal '0XlWG0000000Ihh0AE' with a Get Records on Flow_Environment_Bindings__mdt filtered by DeveloperName='CommSubscription_Marketing'",
    ...
  ],
  "extractedLiterals": [...first 30 string literals from picklist extractor...],
  "note": "v0.4 suggests only. Actual CMT creation + flow refactor lands in v0.5."
}
```

Useful as a planning document. Paste into a ticket. Use it as the blueprint for the refactor.

## v0.5 plan

- One-click "Create CMT records in this org" - POSTs each suggested record to `/sobjects/Flow_Environment_Bindings__mdt/` (Tooling API supports CMT records via REST in newer versions)
- One-click "Refactor flow to use CMT lookups" - adds the Get Records + Variable + reference rewrites to the flow JSON, applies via Tooling PATCH
- Per-environment record manager - view all records, override values, export

These are stretch features. The schema design is the harder problem; the API plumbing is mechanical once the schema is decided.

## When NOT to use CMT

CMT pattern has costs:

- Flow becomes longer (more elements, more variables)
- Runtime adds a CMT lookup per binding (caching helps but is not zero-cost)
- More moving parts to debug
- CMT record creation is a manual or scripted bootstrap per env

For flows that ship to one org and stay there, the cost outweighs the benefit. CMT is for flows that exist in multiple orgs and need stable cross-env behaviour.

The litmus test: if you deploy this flow to >1 org and have hit Id rot at least once, CMT is worth it. If you deploy it once and never touch it, skip CMT.

## References

- [Custom Metadata Types overview](https://help.salesforce.com/s/articleView?id=sf.custommetadatatypes_about.htm)
- [Deploying CMT records via Metadata API](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_custommetadata.htm)
- [Custom Metadata Type loader patterns](https://github.com/forcedotcom/CustomMetadataLoader)
- [Flow record lookup element](https://help.salesforce.com/s/articleView?id=sf.flow_ref_elements_data_lookup.htm)
