# Case study 01: General Enquiry Form Handler - the originating deploy failure

The incident that triggered building the Flow ID Reveal extension. May 2026, a Salesforce sandbox-to-sandbox deploy of a Marketing Cloud Growth Form Submission flow failed with the generic "An unexpected error occurred" message and no actionable detail.

Names and Ids in this case study are real (extracted from the source repo's commits and the actual deploy artifacts). Specifics are intentional because the extension was built to handle exactly this incident class.

## Context

- **Source org:** SilverChef Omni sandbox
- **Target org:** A downstream UAT sandbox (different pod)
- **Flow:** `General Enquiry - AU Form Handler v1 Flow` (label), DefinitionId `300WG00000siVKLYA2`
- **Deploy artifact:** validation deploy `0AfWF00000Dr5QH` via the team's pipeline ("Pipelines Admin")
- **Date:** 2026-05-27
- **Deploy result:** Validation Failed. 1 component error: `flow_5kWq9X89MbMqmcFm` - "An unexpected error occurred. Please include this ErrorId if you contact support: 427833057-64155 (-78598139)"

## What the error looked like

```
Deployment Details
Validation Failed
  Number of Files: 3
  Total Unzipped Size: 26,100 bytes

Component Errors:
  API Name: flow_5kWq9X89MbMqmcFm
  Type:     Flow Version
  Line:     0
  Column:   0
  Error Message: An unexpected error occurred. Please include this ErrorId
                 if you contact support: 427833057-64155 (-78598139)
```

No line. No column. No field. No reference text. The most opaque possible error.

## First-pass diagnosis (without the extension)

The flow XML was retrieved manually. The .flow-meta.xml had 489 lines. The hardcoded Ids hiding in there:

```xml
<actionCalls>
  <name>Email_Consent</name>
  ...
  <inputParameters>
    <name>consentRequest</name>
    <value>
      <apexValue>{"recordList":[{
        "engagementChannelTypeId": "0eFWG0000002ABN2A2",
        "commSubscriptionId": "0XlWG0000000Ihh0AE",
        "commSubscriptionChannelTypeId": "0eBWG00000006rp2AA",
        ...
      }]}</apexValue>
    </value>
  </inputParameters>
</actionCalls>
```

Plus on line 475:

```xml
<form>marketing--Default_Content_Workspace.sfdc_cms__formHandler--MCP4JHBDDOEBHHJKMUYCEKF4YHGY</form>
```

Three sObject Ids + one CMS content key. All from SilverChef (pod code `WG`). Target was on pod `WF`. Salesforce had no records with those Ids in target. Salesforce silently dropped the deploy with the unhelpful error.

Time spent identifying the Ids: about 30 minutes of XML grepping plus reading.
Time spent verifying each Id existed in source: 15 minutes (URL-bar pasting in source SF tabs).
Time spent finding the equivalent target records: indeterminate - different sandbox, different team's CMS setup.

## The light-bulb moment

The realization that motivated building the extension:

> "Flow Builder showed `Email`, `Marketing`, `Marketing Emails` as plain text labels. The Ids were never visible in the UI. They got baked into the metadata at save time. Every flow with picklist-bound objects is at risk. Anyone migrating a Marketing Cloud Growth flow between orgs hits this."

The extension's Scan tab + Compare tab were designed to surface and fix this in seconds rather than hours.

## What we built first

Hours after the failed deploy, the extension v0.1 was scaffolded:

- DOM scan with shadow-root traversal
- Tooling API fetch of flow metadata
- Salesforce Id prefix classifier (3-char prefix → object type)
- Side panel listing every Id with its host JSON path

Total time from "let's build this" to v0.1: about 3 hours.

By v0.3 we'd added the cross-org Compare. By v0.4 the Patch tab (Repoint with Tooling API PATCH). By v0.5 the Cross-org MDAPI deploy. The exact incident that triggered building the extension was solvable in 5 minutes once v0.5 shipped.

## The lessons

### 1. Salesforce error messages do not name the failing reference

The Salesforce deploy validator catches the Id reference failure but the error response strips reference context. This is a Salesforce design choice. Workaround: extract the same information by walking the flow metadata yourself.

### 2. Cross-org Id rot is invisible until deploy time

Flow Builder showing friendly labels prevents anyone from noticing the rot until the deploy fails. There is no warning at flow-edit time that the Id will not survive a cross-org move. This is a Salesforce UX choice.

### 3. The fix is mechanical and repeatable

For a given source-target pair, the same Ids need swapping every time. A persistent Id map (the extension's Org-pair maps feature) means once you've figured out the mapping for one deploy, you don't redo the work for the next.

### 4. CMS forms are a separate layer of rot

Three sObject Ids + one CMS content key = four cross-org-dependent values. Anyone deploying form-triggered flows must handle both. The CMS Form helper in the Inspect tab addresses the CMS layer.

### 5. Production deploys need pre-deploy validateOnly

Running validateOnly catches the rot before a real deploy. Audit → Pre-deploy gives this in 30 seconds. Skipping it means the rot is discovered when the pipeline fails, which is 10x more painful.

## What changed in the extension as a result

This case is the reason for:

- Scan tab's Tooling API path (DOM scan alone could not surface the apexValue-embedded Ids)
- Compare tab's per-Id check (need to know which Ids exist in target before patch)
- Compare tab's Clone → target (because half the missing Ids needed to be created in target anyway)
- Patch tab's Repoint + Cross-org deploy (because Gearset and Change Set won't pick a Draft, see Chapter 12)
- Inspect tab's CMS Form helper (because CMS content keys are the last 10% of the puzzle)
- Audit tab's Pre-deploy (because never again should a deploy fail opaquely without a dry-run first)

Every chapter in this guide traces back to this incident in some way.

## How the same incident plays out with the extension

```
1. Open SilverChef sandbox flow in Flow Builder
2. Extension → Scan → see all 3 Ids + CMS form key in 2 seconds
3. Open UAT sandbox tab
4. Extension → Compare → Per-Id → 3 MISSING + form key (separate handling)
5. Extension → Compare → Clone → target × 3 → 3 new UAT Ids autofill
6. Manual: publish CMS form in UAT, get new content key
7. Extension → CMS Form helper → paste key → Patch
8. Extension → Patch → Apply (or Cross-org deploy)
9. Extension → Lifecycle → Activate in UAT
10. Form submissions work end-to-end

Total time: 15-25 minutes including the manual CMS publish.
```

vs. the original incident which took hours and resulted in pipeline failure.

## What would still go wrong

The extension does not fix everything. In a hypothetical replay of this incident, you might still hit:

- CMS form has different field schema in target → manual schema sync needed
- Custom fields on Prospect object differ between sandboxes → field deploy needed first
- Permission set mismatch → grant access in target before activation
- Required-field validation on cloned record → error surfaces during Clone → manual fix

The extension shortens the diagnosis time and automates the mechanical fixes. It does not eliminate the deploy work entirely.

## References

- Original deploy failure: validation deploy `0AfWF00000Dr5QH`, 2026-05-27
- Flow file: `force-app/main/default/flows/flow_5kWq9X89MbMqmcFm.flow-meta.xml` in the SilverChefOmni repo
- Originating commit triggering the extension build: see the Flow ID Reveal repo for v0.1 initial scaffold
