# 00. Quickstart

Fifteen minutes to install the extension, scan a flow, resolve hardcoded record Ids to readable names, and see your first cross-org Id rot.

## Prerequisites

- Chrome or Chromium-based browser (Edge, Brave, Arc) on macOS, Windows, or Linux
- A Salesforce sandbox you can log into via the browser. Anything works for the install; Form Submission flows are the most interesting target
- The extension source: clone or download `github.com/ialameh/flow-id-reveal` (private repo, ask for access if needed)

## Install in developer mode

1. Open `chrome://extensions/`
2. Toggle **Developer mode** on (top right)
3. Click **Load unpacked**
4. Select the `flow-id-reveal/` folder (the one containing `manifest.json`)
5. Pin the extension to the toolbar (puzzle icon, then pin)

The toolbar shows a blue magnifying-glass-over-a-node icon.

## First scan

1. Open a Salesforce sandbox tab. Log in.
2. Open any flow in Flow Builder. URL pattern: `https://<your-domain>.my.salesforce.com/builder_platform_interaction/flowBuilder.app?flowId=301...`
3. Click the extension icon → **Open side panel**
4. Click **Scan active SF tab**

Within a second, the counts line shows something like:

```
Total 4 | flow metadata: 3 | DOM: 1 | <Flow Label> (301WG00001DY5RJYA1)
```

The rows below list each Id with its prefix-derived object type (`CommSubscription`, `EngagementChannelType`, etc.), an empty Name column, and a host description showing the JSON path where the Id was found.

## Resolve names

Click **Resolve all found Ids**.

The Name column fills in. For a Marketing Cloud Growth form-triggered flow, expect:

| Id | Object | Name |
|---|---|---|
| `0XlWG0000000Ihh0AE` | CommSubscription | Marketing |
| `0eBWG00000006rp2AA` | CommSubscriptionChannelType | Marketing Emails |
| `0eFWG0000002ABN2A2` | EngagementChannelType | Email |

The extension resolved each Id by calling the org's REST sobject endpoint using your session cookie.

## See the cross-org rot

This is where the extension earns its name.

1. In the same Chrome window, open a different sandbox tab. Log in there.
2. Side panel → **Compare** tab → **Per-Id check** → **Refresh tab list**
3. Click the row for your target sandbox in the tab list (border turns blue)
4. Click **Compare against selected target**

The compare table shows, for each Id:

- **EXISTS** in green — the same Id resolves in target. Either same record or different record with that Id.
- **MISSING (404)** in red — the Id does not exist in target. Deploy will fail.
- **ERROR** in amber — something else went wrong (permissions, API version, etc.)

If any Id is MISSING, your flow will not deploy cleanly to that target org. You now know which records to fix.

## Three paths to fix

The **Action** column gives you three buttons:

- **Open source** — opens the record in source org
- **Open target** — opens the record in target org (if it exists)
- **Use this Id** — when EXISTS, autofills the Repoint table with target's Id
- **Clone → target** — when MISSING, auto-creates the record in target by reading source + POST to target via REST + stripping system fields + describe-filtering to createable fields. Autofills the new Id into Repoint.

## What you have now

You have proven the cross-org Id rot is real on at least one flow. You have a working Repoint table with target-org-correct Ids per source Id. You have not patched anything yet.

The next chapters cover:

- The full mental model behind why this happens ([Chapter 01](01-mental-model.md))
- How the extension is architected ([Chapter 02](02-architecture.md))
- The deploy paths once Ids are repointed ([Chapter 12](12-cross-org-deploy.md))

## References

- [GitHub repo (private)](https://github.com/ialameh/flow-id-reveal)
- [Manifest V3 documentation](https://developer.chrome.com/docs/extensions/develop/migrate/what-is-mv3)
- [Salesforce Tooling API reference](https://developer.salesforce.com/docs/atlas.en-us.api_tooling.meta/api_tooling/)
