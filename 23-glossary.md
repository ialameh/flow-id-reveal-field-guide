# 23. Glossary

A-to-Z reference of terms used throughout this guide.

## A

**Active version** — The flow version Salesforce executes when the flow triggers. At most one per flow definition. Immutable; cannot be patched via Tooling API.

**Activate / Deactivate** — Operations that change a flow's ActiveVersionId. Activate sets a specific version as Active. Deactivate clears the Active version (no version fires).

**API version** — Numeric identifier for a Salesforce API surface (e.g. `62.0`). Each version supports different sObjects, fields, and metadata elements. The extension auto-detects the highest version supported by the org.

**Apex action / Apex invocable** — Apex class method exposed to flows via `@InvocableMethod`. Called by `actionCalls` in flow metadata.

**Aura** — Older Salesforce JavaScript framework. Most LEX UI moved to LWC. Some legacy pages remain Aura. The extension's content script can read both.

## B

**Background service worker** — The Manifest V3 background script. Owns cookies, storage, and Salesforce API calls. The trust boundary for the extension.

**Backup** — Export tab feature that copies flow metadata to clipboard. Manual safety net before any Patch.

## C

**Change Set** — Salesforce-native cross-org deploy mechanism. Free, sandbox-or-sibling-only, version-Active-only.

**checkOnly** — MDAPI deploy flag. When true, runs the deploy validator without writing. The Audit → Pre-deploy tab uses this.

**Clone → target** — Compare tab button. Reads a missing record from source, POSTs equivalent to target, returns the new target Id.

**CMS** — Salesforce Content Management System. Stores articles, forms, images. Used by Marketing Cloud Growth Form Submission flows.

**CMS content key** — 28-character GUID identifying a CMS content item. Org-specific. Not a sObject record Id.

**CMS workspace** — Container for CMS content items. Referenced by name in form-triggered flow bindings.

**Communication Subscription (CommSubscription)** — Marketing Cloud Growth object representing a subscription a contact can opt in/out of. Standard sObject, prefix `0Xl`.

**Communication Subscription Channel Type (CommSubscriptionChannelType)** — Links a CommSubscription to an EngagementChannelType. Prefix `0eB`.

**Content script** — Extension component that runs in the Salesforce page context. Performs DOM scan + Tooling API proxy.

**Cookies (chrome.cookies API)** — Extension API for reading HttpOnly cookies. Required for sid retrieval. Background-only.

**CORS** — Cross-Origin Resource Sharing. Browser security policy that blocks cross-origin fetches without proper headers. Background scripts with host_permissions bypass CORS.

**createConsent action** — Marketing Cloud Growth invocable action that writes consent records. Introduced in API v61. Requires v61+ endpoint.

**Custom Metadata Type (CMT)** — Metadata-as-records pattern. Deploys via MDAPI. The long-term fix for cross-org Id rot — see Chapter 17.

## D

**Data Cloud** — Salesforce data platform (formerly CDP). Includes objects like DataLakeObject, DataModelObject, plus CommSubscription chain.

**Deactivate** — See Activate / Deactivate.

**Deploy** — Move metadata from one org to another. Via Gearset, sfdx, Change Set, or extension Cross-org deploy.

**deployment connection** — Salesforce-side configuration that enables Change Set inbound/outbound between two orgs.

**Description (flow)** — Human-readable text on the flow / version. Visible in Setup → Flows. Use to flag staging Drafts.

**Draft** — A flow version that is not Active. Mutable via Tooling API PATCH. Can be edited in Flow Builder.

**Draft collision** — Salesforce error fired when trying to create / receive a second Draft for a flow whose form binding already has a Draft.

## E

**Engagement Channel Type (EngagementChannelType)** — Marketing Cloud Growth object representing a communication channel (Email, SMS, etc.). Prefix `0eF`.

**EngagementChannelType.ContactPointType** — Field denoting whether the channel is Email, SMS, etc.

**Extension** — The Flow ID Reveal Chrome extension.

## F

**Field-Level Security (FLS)** — Per-profile field access controls. The extension respects FLS (you only see/modify fields your profile allows).

**Flow Builder** — Salesforce's visual flow editor. URL pattern `/builder_platform_interaction/flowBuilder.app?flowId=...`.

**Flow Definition / FlowDefinition** — Container for all versions of a flow. Prefix `300`. Holds ActiveVersionId.

**Flow (sObject)** — Tooling API object representing a specific flow version. Prefix `301`.

**FormSubmissionEvent** — Trigger type for flows that fire on CMS form submission. Stored in flow's Start element `triggerType`.

**FullName** — The flow's API name (DeveloperName). Used by MDAPI for cross-org routing. Independent of record Id.

## G

**Gearset** — Third-party Salesforce DevOps tool. Cross-tenant. Handles version selection. Paid.

**Get Records** — Flow element that queries records by criteria. Used in the Auto-pin-to-CMT pattern to look up env-specific Ids.

## H

**Host permissions** — Manifest field listing domains the extension can access. Bypasses CORS for those hosts when fetching from background.

**HttpOnly cookie** — Cookie not readable by page JavaScript via document.cookie. Salesforce sid cookies are HttpOnly. chrome.cookies in background can still read them.

## I

**ID rot** — The cross-org problem this extension addresses. Hardcoded record Ids in flow metadata are pod-specific and break when the flow is deployed to a different org.

**Id (Salesforce)** — 15 or 18 character alphanumeric record identifier. Org-specific. Sequential per pod.

**InvalidDraft** — Flow version status indicating the Draft has validation errors that block save / activation.

## J

**JSON diff** — Line-based diff of pretty-printed JSON. Used by Compare → Cross-org JSON diff and Version diff.

## L

**LCS (Longest Common Subsequence)** — Algorithm used in the extension's diff implementation.

**Lifecycle (Patch tab)** — Subpanel with Activate / Deactivate / Delete buttons.

**Lightning Experience (LEX)** — Salesforce's modern UI. Hosted at `*.lightning.force.com`.

**LWC (Lightning Web Components)** — Salesforce's modern UI framework. Built on web standards including shadow DOM. Flow Builder is LWC.

## M

**Manifest V3 (MV3)** — Current Chrome extension manifest format. Mandatory for new extensions. Uses service worker instead of persistent background page.

**Marketing Cloud Growth** — Salesforce's marketing automation product. Uses Form Submission flows + Data Cloud.

**Metadata API** — Salesforce API for deploying / retrieving metadata via SOAP. Used by Audit → Pre-deploy and Patch → Cross-org deploy.

**MDAPI** — Common abbreviation for Metadata API.

**My Domain** — Salesforce custom domain feature. Affects URL patterns; the extension supports `*.my.salesforce.com`.

**my.salesforce.com** — The host that serves the Salesforce API endpoints. Different from the lightning.force.com UI host.

## O

**Obsolete** — Flow version status. Versions become Obsolete when superseded by a newer Active version.

**Open shadow root** — Shadow DOM whose contents are accessible from outside. LWC uses open shadow roots by default. The extension's walker traverses open roots.

**Org-pair maps** — Persistent Id mapping store. Key is a user-chosen name (e.g. "silverchef-uat"), value is a source-Id → target-Id dictionary.

**originHost** — String passed to background API handlers identifying which org to operate against. Derived from the active SF tab's hostname.

## P

**package.xml** — MDAPI deploy manifest. Lists what is in the deploy package.

**Patch** — Tooling API PATCH on Flow.Metadata. Replaces the metadata of a Draft version in place.

**Pod** — Database instance. Pod code is in characters 3-5 of a Salesforce sObject Id. Same pod = same database = same Ids for records present at last refresh.

**Poisoned Draft** — A source-org Draft after Apply patch, containing target-org Ids. Safe to deploy to target. Unsafe to activate in source.

**Pre-deploy** — Audit tab feature. MDAPI deploy validateOnly against the active org. Dry-run that surfaces deploy errors.

## R

**Repoint** — Patch tab subpanel. The Id-swap table. Source Ids on the left, target Ids on the right.

**REST API** — Salesforce's REST API at `/services/data/v<N>.0/sobjects/...`. Used for sobject CRUD.

**Restore** — Export tab feature. Writes a backup's metadata back into the current flow via Tooling PATCH.

## S

**Salesforce CMS** — See CMS.

**Scan** — The primary extension operation. Surfaces Ids from DOM + Tooling API metadata.

**Search / Replace** — Patch tab subpanel. Generic string find-replace on flow metadata.

**sObject** — Salesforce object type. Each has a 3-char Id prefix.

**Service worker** — MV3 background context. Event-driven, unloaded when idle.

**sid** — Salesforce session token. HttpOnly cookie on `*.my.salesforce.com`. Used as Bearer auth on REST/Tooling/MDAPI calls.

**Side panel** — Chrome's docked panel UI. The extension's main interface.

**Shadow DOM** — Web standard for encapsulated DOM trees. LWC components use shadow DOM. Required walker logic in the content script.

**SOAP** — XML-based RPC protocol. Salesforce Metadata API uses SOAP.

**Source viewer** — Inspect tab subpanel. Renders `state.flowMetadata` as pretty-printed JSON with search highlighting.

**Subflow** — Flow element that invokes another flow. Referenced by FullName in `subflows[].flowName`.

## T

**Tooling API** — Salesforce API for development tools (queries, metadata read/write). Used by the extension for Flow.Metadata operations.

**Trigger order** — The deterministic sequence Salesforce uses to run automation on save (workflow rules, processes, triggers, flows). Audit tab feature lists competing automations.

## U

**Use this Id** — Compare tab button on EXISTS rows. Autofills the Repoint table with the target Id.

## V

**validateOnly** — MDAPI deploy flag. When true, runs validator without writing. Same as checkOnly. Synonyms.

**VersionNumber** — Numeric version of a flow. Increments per save-as-new-version.

**Visualforce** — Older Salesforce UI framework. Not directly used by the extension but present in some Setup pages.

## W

**Workspace** — See CMS workspace.

**walkAll** — Content script generator that traverses every open shadow root + iframe.

## X

**XML escape** — Replacing `&`, `<`, `>`, `"` with entity references. Used in the JSON-to-XML serializer.

## Y / Z

(reserved)

## sObject prefix quick reference

| Prefix | Object |
|---|---|
| `001` | Account |
| `003` | Contact |
| `005` | User |
| `006` | Opportunity |
| `00D` | Organization |
| `00Q` | Lead |
| `00T` | Task |
| `01p` | ApexClass |
| `01q` | ApexTrigger |
| `0AB` | FlowInterview |
| `0Af` | DeployRequest |
| `0DM` | Group |
| `0OD` | DataModelObject |
| `0OE` | DataLakeObject |
| `0PS` | PermissionSet |
| `0Pj` | PartyConsent |
| `0VK` | ManagedContentChannel |
| `0VP` | ContactPointConsent |
| `0WW` | ManagedContent |
| `0Xl` | CommSubscription |
| `0eB` | CommSubscriptionChannelType |
| `0eF` | EngagementChannelType |
| `300` | FlowDefinition |
| `301` | Flow (specific version) |
| `500` | Case |
| `Mzc` | ManagedContentSpace |

See [Chapter 05](05-id-taxonomy.md) for the full taxonomy.
