# Privacy Policy

**Effective:** 2026-05-28
**Last updated:** 2026-05-28
**Applies to:** Flow ID Reveal Chrome extension (all versions)

## Summary

Flow ID Reveal does not collect, transmit, sell, share, or store any user data on any remote server. All processing happens locally in the user's browser. The extension communicates only with the Salesforce organisations the user is actively logged into. No analytics, no telemetry, no third-party endpoints.

## What data the extension accesses

### Salesforce session cookies

The extension reads the user's Salesforce session cookie (`sid`) from `*.my.salesforce.com` and `*.lightning.force.com` domains via Chrome's `chrome.cookies` API. This cookie is used solely as the `Authorization: Bearer` token on REST, Tooling, and Metadata API calls back to the user's own Salesforce organisations. The cookie value is never stored by the extension and is never transmitted to any destination other than the originating Salesforce domain.

### Salesforce metadata and record data

When the user invokes a feature (Scan, Resolve, Compare, Clone, Patch, Deploy, etc.), the extension fetches the relevant flow metadata or record data from the Salesforce API. This data is held in browser memory for the duration of the side panel session. It is displayed in the extension UI and may be copied to the user's clipboard by user action (Export, Backup). It is not transmitted to any third party.

### Browser tab information

The extension reads tab URLs and titles via `chrome.tabs.query` to identify open Salesforce tabs. This information is used in-session only for the cross-org comparison features. Not stored, not transmitted externally.

### Local storage (chrome.storage.local)

Two values are persisted in Chrome's local storage:

- `fir.idMaps`: user-saved Id mapping pairs between source and target organisations (only Salesforce record Ids, no record content)
- `fir.apiVersion`: optional user-specified API version override (a string like "v62.0")

This data resides exclusively on the user's machine. It is never synced to Google's cloud, never transmitted, and is cleared when the user uninstalls the extension.

## What data the extension does not access

- No browsing history outside Salesforce domains
- No bookmarks, downloads, or files
- No location data, geolocation, or device identifiers
- No payment information
- No personally identifiable information beyond what is already inside the user's Salesforce records
- No data from other Chrome extensions

## What data the extension transmits

- **To the user's Salesforce organisations only**: authenticated REST, Tooling, and Metadata API calls. These are the same calls that any logged-in user could make via the Salesforce CLI or any Salesforce client tool. The extension is functionally a UI on top of standard Salesforce APIs.
- **To anywhere else**: nothing.

The extension has no remote server. There is no Flow ID Reveal backend. There is no analytics service. There is no telemetry endpoint.

## Third-party services

None. The extension does not integrate with any third-party service, library, or SDK that performs network communication.

## Data retention

- **Session memory**: held for the duration of the browser session; cleared when the side panel or browser closes
- **Local storage**: persists across browser sessions; cleared when the user uninstalls the extension or clicks "Clear all" in the extension's maps subpanel

The extension does not retain any data on any system other than the user's own browser.

## Data sharing

The extension does not share any data with any third party. The extension does not have the technical capability to share data, because it has no remote endpoint.

## User control

Users have full control over all extension data:

- **Uninstall**: removes the extension and clears all extension-managed local storage
- **Clear maps**: the extension's Patch → Org-pair maps subpanel includes a "Clear all" button that wipes saved Id mappings
- **Clear API version override**: the side panel header includes a "Clear" button for the version override
- **Browser DevTools**: users can inspect chrome.storage contents at any time via the extension's background service worker DevTools

## Permissions and their justifications

The extension requests the following Chrome permissions. Each is used solely as described.

| Permission | Use |
|---|---|
| `cookies` | Read the user's Salesforce sid cookie to authenticate API calls. Restricted to Salesforce host domains. |
| `tabs` | Discover open Salesforce tabs for the cross-org comparison features. |
| `storage` | Persist user-saved Id mapping pairs and API version override locally. |
| `sidePanel` | Display the extension's main UI in Chrome's side panel. |
| `scripting` | Inject a single `console.log` call for the popup's diagnostic feature. |
| `activeTab` | Operate on the active tab after toolbar icon click without per-site prompts. |
| `host_permissions` (`*.lightning.force.com`, `*.my.salesforce.com`, `*.force.com`, `*.salesforce.com`, `*.salesforce-setup.com`, `*.cloudforce.com`) | Make authenticated API calls to the user's own Salesforce organisations from the background service worker. |

No permissions are used for any purpose other than those listed.

## Salesforce-side responsibilities

The extension respects all Salesforce permissions, sharing rules, Field-Level Security, profile restrictions, and organisation-wide settings. It does not bypass any access control. Users can only read or modify Salesforce data that their own logged-in session is authorised to access.

Salesforce data handling, retention, and compliance are governed by the user's Salesforce contract with Salesforce and by the user's organisation policies. The extension does not modify those policies.

## Compliance

The extension itself collects no personally identifiable information and has no compliance footprint. Compliance obligations (GDPR, HIPAA, SOX, etc.) attached to the Salesforce data the user accesses through the extension are the user's responsibility, identical to using any other Salesforce client tool.

## Children's privacy

The extension is a Salesforce developer tool. It is not directed at children under 13 and does not knowingly collect data from any user, including children.

## Changes to this policy

Material changes to this policy will be reflected by updating the "Last updated" date at the top of this page. Users are encouraged to review periodically. The version-controlled history of this policy is available in the extension's documentation repository at https://github.com/ialameh/flow-id-reveal-field-guide/commits/main/privacy.md.

## Contact

For privacy questions, security concerns, or data subject requests:

- **Issues**: https://github.com/ialameh/flow-id-reveal/issues
- **Publisher**: Sam Alameh

## Open source

The extension is source-available. Anyone may audit the code to verify the claims in this policy. Source repository: https://github.com/ialameh/flow-id-reveal.
