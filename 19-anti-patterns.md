# 19. Anti-patterns

What not to do, and why each shortcut backfires.

## 1. Activating the poisoned Draft in source

After Apply patch, the source Draft contains target-org Ids. Activating it in source replaces the working Active version with one that fails on every form submission.

Why this happens:

- "I patched it so it must be ready" misread
- Someone unfamiliar with the workflow opens the flow and clicks Activate to "ship the change"
- Automated workflow tools (DevOps Center, etc.) seeing a Draft and proposing activation

Defenses:

- The extension warns in the Apply confirm dialog. Read it.
- Rename the Draft to "STAGING - DO NOT ACTIVATE". Visible in Setup → Flows.
- Notify the team via Slack / ticket while a poisoned Draft exists.
- Use Cross-org deploy instead, which leaves source untouched.

## 2. Apply patch without target-Id validation

Repoint inputs accept any 15/18-char alphanumeric. There is no pre-Apply check that the pasted Ids actually exist in any org.

What goes wrong:

- Typo: pasted `0XlWFOOOOOOOGrB0AU` (capital O instead of zero) → patches with invalid Id → Salesforce accepts the PATCH but the flow at runtime hits 404.
- Wrong org's Id: pasted UAT Id while patching for prod deploy → flow tries to use UAT Ids in prod → runtime failure in prod.

Defense:

- Use **Use this Id** (after Compare → EXISTS) or **Clone → target** (after Compare → MISSING) instead of manual paste. These autofill validated Ids.
- For manual paste, run Compare → Per-Id check first to verify the target Ids exist.
- v0.6 candidate: pre-Apply validation that pings every pasted Id against the active org's REST endpoint.

## 3. Scanning Active and trying to PATCH

The flow you scanned is V7 Active. You fill Repoint. Click Apply. Salesforce rejects:

```
Cannot modify Flow because it is not a Draft
```

Active versions are immutable. PATCH on Active returns an error. The extension blocks early with a clear message when `state.flowRecord.Status !== "Draft"`, but if Status reads as `Draft` due to scan staleness while the actual org state is `Active`, the API rejects.

Defense:

- Navigate to the Draft version in Flow Builder. URL has flowId of the Draft, e.g. `?flowId=301WG00001DXvoVYAT`.
- Re-Scan after switching versions.
- Use Cross-org deploy instead - it creates a new Draft in target regardless of source's Active/Draft status.

## 4. Treating the extension as a deploy tool

The extension is for cross-org Id management, not multi-component deployment. Use it alongside Gearset / Change Sets / sfdx, not instead of them.

What goes wrong if you treat it as the deploy tool:

- Multi-component deploys (Apex + Flow + Custom Field) need all-or-nothing transactional deploy. The extension does one Flow at a time.
- No CI integration. Browser-bound only.
- No audit trail. Extension actions don't write to a deploy log accessible to ops.

Defense:

- For production deploys, run Gearset / sfdx / Change Set with proper change management.
- Use the extension to fix the flow contents before the deploy.
- Use Audit → Pre-deploy as a pre-flight check against the production validator.

## 5. Forgetting to back up before Apply

Apply rewrites the source Draft. No undo button.

What goes wrong:

- A typo in Repoint values writes garbage Ids to the Draft.
- A network error mid-PATCH leaves the Draft in an undefined state.
- A confirm-dialog mis-click activates the wrong version.

Defense:

- Always click Export → Backup current flow metadata to clipboard before Apply.
- Save the backup JSON to a file (not just the clipboard).
- Restore via Export → Restore from clipboard if needed.
- Use Patch → Cross-org deploy validate-only first; if validate fails, no source mutation happened.

## 6. Skipping Compare before Patch

You scanned source. You assume target has the same records with the same Ids. You fill Repoint with the source Ids you saw. Click Apply.

Source Draft now has source Ids. No change. You deploy. Deploy fails.

Or worse: you assume target has DIFFERENT Ids and paste random target Ids without verifying. Source Draft gets poisoned with wrong Ids.

Defense:

- Always run Compare → Per-Id check before Repoint. The check tells you which Ids exist and what the target's equivalents are.
- Use Use this Id / Clone → target buttons to autofill with verified target Ids.

## 7. Not checking Draft collision

You patch source. Run Cross-org deploy. Deploy fails:

```
The form you selected is already associated with a draft flow version
```

Target had a leftover Draft from a previous attempt. You did not check.

Defense:

- Compare → Draft collision check before any deploy. One-click verdict.
- If BLOCKED, handle target's existing Draft first (Activate or Delete via Lifecycle in the target tab).

## 8. Mixing API versions mid-workflow

You scanned with API v60 auto-detected. You set the override to v62. You ran Apply. You set override back to auto (which redetects to v60 cached). The next Cross-org deploy uses v60 again.

What goes wrong: validateOnly results aren't reproducible. Errors that appeared in v60 may not appear in v62. Confusion ensues.

Defense:

- Pick one version for a workflow. Don't change mid-flight.
- If you must change, redetect explicitly (Auto button) and verify the header status line.

## 9. Editing the extension code while the flow is open

You realised the extension has a bug. You fix `sidepanel.js`. You reload the extension. The side panel is now broken because Chrome closed it on reload.

What goes wrong: state.scanned, state.flowMetadata, state.maps are all in side-panel memory. Reload wipes them.

Defense:

- Backup any in-memory state before editing extension code.
- Save Repoint values to the persistent map first (`Save swaps to map`).
- Export → Backup current flow metadata.
- Re-Scan after the side panel reopens.

## 10. Running scans on production flows during peak hours

Scan is read-only but every Resolve clicks adds a REST API call to the org's API quota. Bulk Resolve all on a flow with 50+ Ids adds 50 calls.

Concrete cost: production orgs have ~15k API calls/24h baseline for older editions. 50 calls is 0.3%. Negligible at low volume, observable at high volume.

Defense:

- Production scans are fine. Just don't loop them.
- Bulk Resolve sparingly.
- Use the Inspect → Source viewer (cached, no API calls beyond the initial scan) for re-inspection.

## 11. Cloning records that should be metadata

You see a MISSING `01p...` (ApexClass) in Compare. Click Clone → target. Extension tries POST to `/sobjects/ApexClass/`. Returns 400 (ApexClass is non-createable via REST).

ApexClass, Profile, PermissionSet, FlowDefinition, and other metadata types are not data. They deploy via Metadata API only.

Defense:

- Use Compare → Create manually link, then deploy the metadata via Gearset / sfdx / Change Set.
- The extension's describe filter usually catches this and the error tells you (`createable: false`).

## 12. Relying on the extension's JSON-to-XML serializer for production deploys

The extension's `metadataJsonToFlowXml` is heuristic. For most flows it produces deploy-acceptable XML. For complex flows with rare elements, it may produce slightly off output that the deploy validator rejects.

Defense:

- For production deploys, use sfdx retrieve to get canonical XML, then deploy that.
- Use the extension's Cross-org deploy for sandboxes and dev iteration only.
- Validate-only first, always.

## 13. Trying to "fix" source by restoring AFTER deploy when target already activated

You deployed the poisoned Draft to target. Activated it in target. Then realised you forgot to restore source.

If you then run Repoint INVERSE in source, source Draft now has source Ids. Fine.

But if you confuse yourself and accidentally run the original Repoint (forward direction) again in source, you re-poison source. Now you have a poisoned source and target is fine - opposite of where you started.

Defense:

- Save swaps to a named map immediately after the first Apply. The name protects you from forward/inverse confusion.
- Always read the Repoint table column headers. Source Id on left, Target Id on right. Prefill INVERSE swaps the meaning, but the inputs visually look the same.
- Take a breath before clicking Apply twice in a session.

## 14. Letting the extension's icons mislead you on tab state

Toolbar shows the extension icon. Side panel shows the SF tab's URL in the header. These do not auto-update if you switch tabs while the side panel is open in another window.

Defense:

- Check the header URL before clicking Scan. If it shows the wrong URL, click the SF tab to refocus.
- Refresh the side panel if state seems out of sync.

## 15. Believing the extension knows what's in a closed shadow root

Salesforce sometimes uses closed shadow roots for sensitive components. The extension's DOM walker cannot traverse closed roots.

Defense:

- For Flow Builder, the Tooling API path bypasses DOM entirely and is the source of truth.
- For non-flow Setup pages, the DOM scan is best-effort. Missing Ids are possible.
- Cross-check via Inspect → Source or direct Tooling API queries when in doubt.

## References

- [Salesforce API limits](https://developer.salesforce.com/docs/atlas.en-us.salesforce_app_limits_cheatsheet.meta/salesforce_app_limits_cheatsheet/salesforce_app_limits_platform_api.htm)
- [Flow activation rules](https://help.salesforce.com/s/articleView?id=sf.flow_distribute_activate.htm)
- [Closed shadow roots and extension limitations](https://developer.chrome.com/docs/extensions/develop/concepts/match-patterns)
