# 18. Troubleshooting

Every error you may see, what it means, what to do.

## Scan returns 0 known Ids

**Counts line shows `Total 0 | flow metadata: 0 | DOM: 0`**

Possible causes and fixes:

| Cause | Fix |
|---|---|
| Content script not loaded (extension just installed) | Cmd+Shift+R on the SF tab to re-inject content script |
| Page not yet ready | Wait 2 seconds, click Scan again |
| Tab is not a Salesforce URL | Open a SF tab; check URL matches manifest host patterns |
| Tooling fetch failed (network blocked) | Check DevTools Console for `[Flow ID Reveal] scanned` log |

**Counts line shows `Total 1 | flow metadata: 0 | DOM: 1`**

You see only the org Id from DOM. Tooling API path failed. Common causes:

| Cause | Fix |
|---|---|
| Not on a Flow Builder URL | Navigate to `?flowId=301...` |
| Session expired | Refresh SF tab to re-auth |
| API version mismatch (auto-detect failed) | Header → click "Auto" to redetect |
| Custom Domain or My Domain misconfigured | Verify `<host>.my.salesforce.com` is reachable in another tab |

## Resolve returns ERR 401 / INVALID_SESSION_ID

```json
[{"errorCode":"INVALID_SESSION_ID","message":"Session expired or invalid"}]
```

Sid cookie is missing or expired. Fix: Cmd+Shift+R on the SF tab to re-auth. The extension reads cookies fresh on every call.

If you are on `*.lightning.force.com` but the extension is trying to call `*.my.salesforce.com`, the cookie may be on a different domain than the extension is reading. Open the my.salesforce.com URL directly in a tab once, log in, then re-scan.

## Resolve returns 404

Record does not exist in that org. Either:

- Source Id is from a different org (you scanned a tab that wasn't the source)
- Record was deleted
- You typed the Id wrong (manual resolve)

For cross-org rot, this is the expected MISSING state on Compare. For same-org resolve, the record is gone.

## Apply patch returns success but Source still has old Ids

Tooling API silent-drop quirk. The PATCH returned 200 OK but Salesforce did not actually persist the change. Known issue for form-bound flows in some API versions.

Workaround: use Patch → Cross-org deploy (validateOnly first, then write) instead. MDAPI deploy returns explicit errors and reliably persists.

Or use Flow Builder UI directly: open the flow, click Save, click Save As New Version. This writes through a different code path.

## Apply patch returns 400 with ACTIONCALL_NOT_ACTIVE_FOR_API_VERSION

```
We couldn't deploy the flow because the <Action Name> action is unsupported
in API version 60. Use API version 61 and later.
```

The flow uses an action introduced in a newer API version than the endpoint is using.

Fix: side panel header → API ver → click "Auto" to redetect, or type a higher version like `v63.0`. Retry Apply.

## Apply patch fails with "FLOW_NOT_DRAFT" or similar

The flow version you scanned is Active or Obsolete. Tooling API PATCH only works on Drafts.

Fix: navigate to the Draft version in Flow Builder. URL pattern includes flowId of the Draft, e.g. `?flowId=301WG00001DXvoVYAT`. Re-Scan, retry Apply.

## Cross-org deploy fails with form binding error

```
The form you selected is already associated with a draft flow version.
You can't associate a form to more than one draft flow version.
```

Target org has an existing Draft for this flow's form. See [Chapter 13](13-form-bound-flow-constraints.md).

Fix: in target tab, side panel → Patch → Lifecycle → Activate or Delete the existing Draft. Then retry Cross-org deploy.

## Cross-org deploy fails with form-not-found

```
The form '<reference>' does not exist
```

Target's CMS does not have a form with the content key the flow references.

Fix: Publish the equivalent form in target's CMS first. See [Chapter 15](15-cms-form-helper.md) for the helper that handles this. Then re-patch the flow's `<form>` reference and retry deploy.

## Clone → target fails with "missing required field"

Target's sobject describe reports a field as required that the source record did not populate.

Fix: read the error, identify the field, add a value to the cloned payload, retry. The extension does not allow per-clone field editing in v0.5.2; workaround is to manually POST via Workbench REST Explorer with the required field included.

## "Browser is already in use for /Users/sam/Library/Caches/ms-playwright/..."

This is a Playwright MCP error, not an extension error. Means another Claude Code session has Playwright Chrome open.

Fix: close the other session, or kill the Chrome process. Not extension-related.

## Side panel does not open

Click extension icon. Click "Open side panel". Nothing happens.

Fix: Chrome MV3 requires the side panel to be opened from a user gesture in a normal tab. If the active tab is `chrome://extensions/` or `chrome:` URL, side panel cannot open. Switch to a Salesforce tab and click again.

## Scan returns the same old Ids after Apply

You verified Apply succeeded but re-Scan still shows old Ids.

Possible: scan refreshed `state.flowMetadata` to a stale Tooling response. Salesforce sometimes serves cached metadata briefly after PATCH.

Fix: wait 5 seconds, re-Scan. Or hard-refresh the SF tab and re-scan.

If persistently stale, use Inspect → Source → search for the new Id prefix. If Source shows new Ids, the metadata IS updated; the scan classifier missed them. File a bug.

## Clone fails with "INSUFFICIENT_ACCESS"

Your target session lacks Create permission on that sObject.

Fix: grant the permission via Profile or Permission Set in target. Verify with sf data create record CLI before retrying via extension.

## Pre-deploy validateOnly returns ENTITY_IS_LOCKED

```
Cannot modify flow because it is being modified by another user
```

Someone else has the flow open in Flow Builder, or your previous Apply is still being processed.

Fix: wait 30 seconds, retry. Or check Setup → Flows → Manage to see if anyone is editing.

## SOAP deploy returns 500 with no body

Network issue or malformed SOAP envelope. The extension's envelope is tested but corner cases exist.

Fix: check DevTools Network tab for the failed POST. Inspect the request body. File a bug with the response status + headers.

## Lifecycle "Activate" fails with "no permission to activate"

Your session lacks "Manage Flow" or "Manage Force.com Flow" system permission.

Fix: grant via Profile or Permission Set. Standard System Administrator profile has it.

## "Could not establish connection. Receiving end does not exist."

Content script is not loaded in the tab the side panel is trying to message.

Common after:

- Just installing the extension (existing tabs don't have the content script)
- Loading a different extension that conflicts
- Chrome decided to evict the service worker

Fix: hard-refresh the SF tab. Reload the extension card if persistent.

## Export Copy JSON / CSV produces empty output

`state.scanned` is empty. Run Scan first.

If Scan was run and the table has rows but Export is empty, this is a bug. File it.

## Compare → Refresh tab list shows no tabs

The tab query returned no SF tabs.

Possible:

- No SF tabs open in any window (open one)
- All SF tabs are in a different Chrome profile (each profile has separate tabs)
- The URL pattern is mismatched (the extension supports lightning.force.com, my.salesforce.com, force.com, salesforce.com, salesforce-setup.com, cloudforce.com — if your org uses a different domain, add it to host_permissions)

## Source viewer is blank after "Load flow source"

`state.flowMetadata` is null. Run Scan on a Flow Builder page first.

If Scan ran but state.flowMetadata is still null, the Tooling fetch failed. See "Scan returns 0 known Ids" above.

## I still cannot figure it out

Open the extension's side panel. Open Chrome DevTools on the side panel (Inspect element on the side panel itself). Check Console for errors.

Also open DevTools on the SF tab. Check Console for `[Flow ID Reveal]` log lines.

Open DevTools on `chrome://extensions/` → Background page → Console for background errors.

File a bug at `github.com/ialameh/flow-id-reveal/issues` with:

- The error text (paste verbatim)
- Steps to reproduce
- API version your org reports (header shows it after detection)
- Which tab / subpanel triggered the error
- Console output from above

## References

- [Tooling API error codes](https://developer.salesforce.com/docs/atlas.en-us.api_tooling.meta/api_tooling/sforce_api_calls_concepts_core_data_objects.htm)
- [Metadata API error codes](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_status_codes.htm)
- [Chrome extension debugging guide](https://developer.chrome.com/docs/extensions/get-started/tutorial/debug)
