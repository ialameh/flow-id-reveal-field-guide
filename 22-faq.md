# 22. FAQ

Quick answers to recurring questions.

## Does this work with sfdx instead of Gearset?

Yes. The extension is deploy-tool-agnostic. It fixes the flow contents before deploy. sfdx, Gearset, Change Sets, and the extension's built-in Cross-org deploy all work with patched flow metadata.

## Can I use this in production orgs?

Read operations (Scan, Resolve, Compare, Inspect, Audit) are safe in production. Write operations (Apply, Cross-org deploy, Lifecycle) are technically possible but require care.

Best practice: validate-only against production from your dev environment, ship via your established CI pipeline (Gearset / sfdx + tests + approvals).

## What if my org uses a custom domain (My Domain) with a weird subdomain?

The manifest's `host_permissions` covers all standard SF domains. Custom domains using `*.my.salesforce.com` work. Custom domains using a fully custom DNS (e.g. `app.acme.com`) require adding that domain to host_permissions and reloading the extension.

## Does this work in Salesforce mobile?

No. The extension runs in Chrome desktop only. Salesforce mobile is a different runtime.

## Can I script this from CI?

No. The extension is browser-bound. For CI, use sfdx, Gearset CLI, or write your own Tooling API client.

The patterns in this guide (Repoint, validate-only, lifecycle) translate to sfdx + script. The extension is the interactive front-end; the same operations are scriptable.

## Will it work on Edge / Brave / Arc?

Yes. They are Chromium-based and support MV3.

## Why is my Scan returning Ids I have not used?

The DOM scan picks up the org Id (`00D...`) from a `<script>` tag on every Lightning page. That's harmless — informational.

If you see unfamiliar `001`, `003`, etc. Ids, they may be sample records linked from the LEX shell (e.g. recently viewed). The flow itself does not necessarily reference them. Check the host column to see where each Id was found.

## Why does the same flow's scan show different Id counts between scans?

Two reasons:

- Flow Builder may have rendered new DOM since the last scan (you clicked into a different action, opening its right panel which spawns more elements)
- The Tooling API may have served slightly different metadata if the flow was re-saved between scans

For deterministic Id counts, use Inspect → Source on the same load.

## Can I share my Org-pair maps with my team?

Yes. Patch → Org-pair maps → Copy JSON, paste into a shared doc / git file. Team members → Import from clipboard.

Or commit the JSON to a config repo and have teammates import. The map format is plain JSON.

## What happens to my Id mappings if I uninstall the extension?

`chrome.storage.local` data is preserved during a normal disable/enable but wiped on uninstall.

Export your maps before uninstall (Patch → Org-pair maps → Copy JSON, save to a file).

## Will the extension auto-update?

When loaded as an unpacked extension, no. You pull updates from the git repo and reload manually.

When packaged and distributed via a private CRX, Chrome can auto-update if the manifest specifies an `update_url`. Not enabled by default.

## How do I add support for a new sObject prefix?

1. Edit `src/utils/id-patterns.js` — add the prefix to `ID_PREFIX_MAP`
2. Edit `src/content.js` — add the same prefix to the mirrored constant
3. Reload the extension
4. Rescan

The two files have a copy of the prefix map because content scripts don't natively support ES modules in MV3.

## Why does my Apply succeed but the change is not visible in Flow Builder UI?

Flow Builder caches the flow metadata aggressively. After a Tooling PATCH, the open Flow Builder tab still shows the pre-patch state until you reload.

Fix: hard-refresh the Flow Builder tab (Cmd+Shift+R). The patched version loads.

## Can I edit flow LOGIC (decisions, actions) via the extension?

No. The extension patches Ids and literals only. For flow logic edits, use Flow Builder UI directly.

You can use Patch → Search/Replace to do mechanical text swaps on any string value in the JSON, including action names or connector targets, but the safer path is Flow Builder for logic changes.

## What if I want to patch a flow's `apiVersion` field?

Patch → Search/Replace. Find `"apiVersion":60.0`, Replace `"apiVersion":62.0`. Apply.

Or edit the metadata via Inspect → Source (manual JSON edit not currently supported in v0.5.2; v0.6 candidate). For now use Search/Replace as the mechanism.

## Does Cross-org deploy run tests?

No. The extension's MDAPI deploy does not specify a test level. Salesforce defaults to RunLocalTests for production, RunSpecifiedTests for sandboxes.

For production deploys you should run all relevant tests. Configure via your CI tool, not via this extension.

## Why is my CMS form not showing in the workspaces UI in target?

CMS forms have visibility scopes. The form may exist but be:

- Unpublished (publish it via the form detail page)
- In a workspace your user does not have access to (grant access via Setup → CMS Workspaces)
- In a different workspace than the flow expects (the flow's reference includes the workspace name)

## What if my target org has the form but on a different content key than what I expect?

Use Inspect → Source on the target org's flow (if you have a flow there) to find the content key. Or query `ManagedContentVersion` directly. Or paste the URL of the form's detail page into the CMS Form helper to extract the key.

The CMS Form helper does not auto-query target's CMS for the matching form. Manual content key fetch is needed.

## How is this different from existing tools like Salesforce Inspector?

Salesforce Inspector is excellent for general metadata + SOQL + record inspection. It does not specialise in cross-org Flow Id management.

This extension is narrower: Flow Builder-aware Id surfacing, cross-org compare, in-extension patching + deploying. It overlaps with Inspector in some areas (sObject record inspection) but the workflow is different.

Use both. They are complementary.

## Will there be a Firefox version?

Firefox supports MV3 with minor differences. A port is doable. Not on the v0.x roadmap.

## Is there a paid version?

No. The extension is free and source-available under the MIT-style license for the extension code. The field guide (this documentation) is CC BY 4.0.

## Where do I report bugs or request features?

`github.com/ialameh/flow-id-reveal/issues` for the extension. The repo is private; ask for read access if needed.

`github.com/ialameh/flow-id-reveal-field-guide/issues` for this documentation.

## Why "Flow ID Reveal"?

Because the extension reveals the hidden record Ids inside Flow Builder. The name describes what it does. Other names considered: "Flow Id Inspector", "Flow Cross-Org", "Flow Dossier". "Reveal" stuck because the original use case was "I cannot see the ID anywhere, please reveal it".

## References

- [Salesforce Inspector](https://chrome.google.com/webstore/detail/salesforce-inspector-relo/hpijlohoihegkfehhibggnkbjhoemldh)
- [ORGanizer for Salesforce](https://chrome.google.com/webstore/detail/organizer-for-salesforce/eopncngiehmnlhpennefdjdgcaibnbej)
- [Salesforce DevOps Center](https://help.salesforce.com/s/articleView?id=sf.devops_center_overview.htm)
- [Salesforce CLI documentation](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference.htm)
