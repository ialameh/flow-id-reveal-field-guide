# 10. Audit tab

Pre-deploy safety + org-wide views. Five subpanels.

## Pre-deploy

MDAPI validateOnly dry-run of the current flow against the active org.

Click **Run pre-deploy validation**.

The extension:

1. Builds an MDAPI deploy package: the current flow's metadata as `.flow-meta.xml` + a `package.xml`, ZIPped in-memory using the inline ZIP writer.
2. Calls Metadata API SOAP `deploy()` with `checkOnly=true`.
3. Receives an `asyncId`.
4. Polls `checkDeployStatus` every 2 seconds for up to 60 seconds.
5. Reports PASSED or FAILED with full error detail.

Catches:

- Schema mismatches (target lacks a referenced field)
- API version conflicts (flow uses a v61+ action on v60-only org)
- Reference resolution failures (Id rot)
- Permission issues
- Form-binding collisions

Same validation that Gearset or Change Set runs. Same error format. Surfaces the issues before the pipeline gets involved.

### Why use this instead of just deploying

Three reasons:

- **Fast iteration.** Fix one issue, re-run validate, see next issue. No need to wait for a pipeline cycle.
- **Org-safe.** validateOnly never writes.
- **Pre-flight.** Catches deploy blockers while the deploy plan is still in your head.

## Perf scan

Click **Scan performance risks**. Tabulates flow patterns known to cause governor limit issues.

Detected today:

| Severity | Issue |
|---|---|
| **HIGH** | DML (Create / Update / Delete) inside a loop |
| **HIGH** | SOQL (Lookup) inside a loop |
| **MED** | Record Create with no fault path |
| **MED** | Record Update with no fault path |
| **LOW** | Action with no fault path |

The "inside loop" detection walks forward from each `loops[].nextValueConnector.targetReference`, stopping at the `noMoreValuesConnector.targetReference`, and collects every node name in that body. Any DML or SOQL node in that set is flagged.

Why these matter:

- DML in loop: hits the 150 DML per transaction limit on bulk data
- SOQL in loop: hits the 100 SOQL per transaction limit
- Missing fault paths: silent failures, no error logging, debugging nightmare

The output table shows severity, issue type, and the offending node name. Click into the flow in Flow Builder to fix.

Limitations:

- Does not detect SOQL-in-loop hidden inside Apex action calls (the action's internals are opaque)
- Does not estimate transaction record counts (just topology, not volume)
- Does not detect inefficient filters or non-selective queries (Salesforce's Query Plan tool does that)

## Trigger order

For record-triggered flows, lists every Apex Trigger + autolaunched flow on the same SObject.

Enter an SObject name (e.g. `Account`). Click **Find competing automation**.

Two queries run:

```sql
-- ApexTrigger
SELECT Id, Name, TableEnumOrId,
       UsageAfterDelete, UsageAfterInsert, UsageAfterUndelete, UsageAfterUpdate,
       UsageBeforeDelete, UsageBeforeInsert, UsageBeforeUpdate, Status
FROM ApexTrigger
WHERE TableEnumOrId = '<sobject>'
```

```sql
-- FlowDefinitionView (autolaunched flows)
SELECT Id, DeveloperName, MasterLabel, TriggerType, Status
FROM FlowDefinitionView
WHERE ProcessType IN ('AutoLaunchedFlow')
```

Output is JSON of both result sets. Flow filter is broad (FlowDefinitionView lacks an SObject filter); the extension does a label-substring match to narrow the flow list. Cross-reference manually if the label naming convention does not include the SObject name.

Useful for:

- Untangling "why did my flow run third" mysteries
- Auditing all automation on a critical object before a major change
- Migration planning when consolidating triggers and flows

## Flow inventory

Click **Load all flows**. Lists every FlowDefinition in the active org.

Columns:

- Label (MasterLabel or DeveloperName)
- Active version number
- Latest version number + Status

Filters:

- **Filter by label** (text input) — substring match
- **Status** dropdown — All, Active only, Draft only, Obsolete only, InvalidDraft only

The counts line below shows `N of M flow definitions` after filtering.

Use cases:

- Audit org for orphan Drafts (`InvalidDraft` filter)
- Find candidates for the next cleanup sprint (`Obsolete` filter)
- Get a count of "how many flows is this org running" for capacity planning

## Reverse Id lookup

Type any Id, find every flow whose Metadata contains it.

Useful for:

- "What flows reference this CommSubscription?"
- Pre-emptive impact analysis before deleting a record
- Finding orphan flow references after a record is deleted

The query is expensive:

```sql
SELECT Id, MasterLabel, VersionNumber, Status, Metadata
FROM Flow
WHERE Status IN ('Active', 'Draft')
```

This pulls the full Metadata blob for every active or draft flow in the org. On a big org with hundreds of flows, this can take 30-60 seconds and return megabytes of JSON.

The extension then walks each Metadata blob locally, checks for substring match on the target Id, and reports hits.

Tradeoffs:

- Full coverage (every active/draft flow)
- Slow on big orgs
- Excludes Obsolete versions (which is usually what you want)

To narrow to specific flows, query a subset first via the Flow Inventory tab, then use the Source viewer's search on each one. Slower per-flow, faster overall on huge orgs.

## What Audit does not do

- Pre-deploy uses an in-extension XML serializer that is heuristic, not authoritative. For complex flows with edge-case elements, the serializer may produce slightly different XML than Salesforce's official MDAPI retrieve. Errors in the validateOnly response may include differences caused by the serializer. Use sfdx retrieve + deploy for the absolute reference.
- Perf scan does not analyze Apex actions or invocable methods (no source visibility from the flow level).
- Trigger order does not show actual execution order (Salesforce's automation order is deterministic but complex; this lists the cast, not the play).
- Reverse Id lookup misses Obsolete flows.

## References

- [Metadata API checkOnly deploy](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_deploy.htm)
- [Order of execution on save](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_triggers_order_of_execution.htm)
- [Governor limits](https://developer.salesforce.com/docs/atlas.en-us.salesforce_app_limits_cheatsheet.meta/salesforce_app_limits_cheatsheet/salesforce_app_limits_platform_apexgov.htm)
- [FlowDefinitionView object](https://developer.salesforce.com/docs/atlas.en-us.api_tooling.meta/api_tooling/tooling_api_objects_flowdefinitionview.htm)
