# 21. Privacy and security posture

What the extension reads, what it sends, where data goes, and what it intentionally does not do.

## TLDR

- No telemetry. No analytics. No remote logging.
- No external network calls. Every request goes to your own Salesforce orgs.
- No credential storage. Session tokens are read live from cookies; nothing persists.
- No data exfiltration. Everything stays in your browser + the orgs you point it at.

## What the extension reads

### From your browser

- **Salesforce session cookies (`sid`)** from `*.my.salesforce.com` and `*.lightning.force.com` domains. Read via `chrome.cookies.get()`. Used as `Authorization: Bearer <sid>` header on REST/Tooling/MDAPI calls. Never stored, never logged, never sent anywhere except as the auth header to Salesforce itself.
- **Tab URLs and titles** via `chrome.tabs.query()`. Used to identify which SF tabs are open. Never stored, never logged.
- **Active tab contents** via content script. Only on URLs matching the manifest's `host_permissions` (Salesforce-related domains).
- **Local storage** via `chrome.storage.local`. The extension reads `fir.idMaps` (Id pair mappings) and `fir.apiVersion` (version override).

### From Salesforce

- **Flow metadata** via Tooling API SOQL.
- **sObject record fields** via REST sobject GET for resolve/clone operations.
- **Org capabilities** via the unauthenticated `/services/data/` endpoint for API version detection.

The extension does not read anything else from Salesforce. It does not query User records to identify you. It does not pull org settings, profile data, or any reporting / analytics data.

## What the extension writes

### To Salesforce (when you explicitly trigger it)

- **PATCH** on `Flow.Metadata` (Tooling API) — Apply patch, CMS Form helper patch, Search/Replace patch
- **PATCH** on `FlowDefinition.Metadata.activeVersionNumber` — Lifecycle Activate / Deactivate
- **DELETE** on `Flow` (Tooling API) — Lifecycle Delete Draft
- **POST** on `<SObject>` (REST API) — Clone → target
- **SOAP** to `/services/Soap/m/<version>` with `deploy()` — Pre-deploy validate, Cross-org deploy

Every write requires an explicit click on a button in the side panel. No silent writes.

### To your browser

- **chrome.storage.local** for org-pair Id maps and API version override. Cleared via the maps "Clear all" button or by uninstalling the extension.

### Nowhere else

No remote servers. No log endpoints. No analytics services. No telemetry beacons.

## What the extension does NOT do

- Does not phone home
- Does not load remote code (no eval of fetched scripts, no remote service worker)
- Does not include external libraries beyond what is in the source tree
- Does not contact any URL outside the SF host_permissions list
- Does not log session tokens anywhere
- Does not persist session tokens
- Does not refresh OAuth tokens
- Does not request OAuth scope expansion
- Does not provide an external API for other extensions to call into

## Required permissions

```json
"permissions": ["activeTab", "scripting", "storage", "sidePanel", "tabs", "cookies"],
"host_permissions": [
  "https://*.lightning.force.com/*",
  "https://*.my.salesforce.com/*",
  "https://*.force.com/*",
  "https://*.salesforce.com/*",
  "https://*.salesforce-setup.com/*",
  "https://*.cloudforce.com/*"
]
```

Per-permission rationale:

| Permission | Why needed |
|---|---|
| `cookies` | Read HttpOnly sid for API auth. Restricted to host_permissions domains. |
| `tabs` | Discover open Salesforce tabs across windows for the cross-org compare. |
| `storage` | Persist Id pair mappings + API version override. |
| `sidePanel` | Open the main UI. |
| `scripting` | Quick-scan path injects a console.log call from the popup. |
| `activeTab` | Operate without per-domain prompts after user clicks the toolbar icon. |
| `host_permissions` | Cross-host fetch to Salesforce API endpoints from background. |

Notably absent:

- `webRequest` / `webRequestBlocking` — the extension does not intercept or modify network traffic outside its own fetches
- `<all_urls>` — only Salesforce hosts are permitted
- `bookmarks`, `history`, `geolocation`, `notifications` — none of these are used

## Data minimisation

The extension uses no more data than needed:

- DOM scan reads only attribute values matching the Salesforce-Id regex pattern. Other DOM content is not extracted.
- Tooling SOQL fetches only the fields the operation needs: `Id, Status, VersionNumber, MasterLabel, Metadata` for Flow, plus DefinitionId-resolution queries.
- sObject GET returns the full record (as the REST API requires for a single-record GET); the extension does not retain it beyond the immediate operation except as Name resolution + Clone source.
- Backup explicitly captures full flow metadata, but only to your clipboard, only on explicit Backup click.

## Cross-org isolation

The extension does not share data between orgs except where you explicitly trigger it (Compare, Clone, Cross-org deploy).

Each org's session is read fresh from its own cookie. The extension does not maintain a session pool. Logging out of one org does not affect the extension's interaction with another.

## What you have to trust

To use the extension you trust:

1. The extension code itself (auditable, source available, ~1500 lines)
2. Salesforce's REST + Tooling + Metadata APIs (industry standard)
3. Chrome's permission model + host_permissions enforcement
4. The npm-free, dependency-free build (no transitive supply chain risk)
5. The local browser sandbox

The extension does not require trusting any third-party server.

## Audit recommendations

Before deploying widely in your organisation:

- Review the source tree (especially `background.js` for the API calls + cookie handling)
- Run the extension in a personal sandbox first
- Check the manifest matches what is documented here
- Monitor Network tab during typical operations to confirm requests only go to your orgs
- Pin to a specific git commit hash; do not auto-update past audited versions

## Threat model considerations

| Threat | Mitigation |
|---|---|
| Malicious script in a Salesforce page injects fake Ids | DOM scan is read-only; consequences limited to confused UI. Apply requires user click. |
| Compromised Chrome profile reads session cookies | Out of scope — Chrome profile compromise affects all your auth |
| Compromised extension code (supply chain) | Pin to known-good commit hash. Audit code on each upgrade. |
| Server-side Salesforce vulnerability | Out of scope — affects all Salesforce API consumers |
| Accidental cross-org write (e.g. Apply on production by mistake) | Always read confirm dialogs. Use validate-only for cross-org deploy first. |

## Compliance

The extension itself has no compliance footprint (no data collection, no transmission). The Salesforce orgs you operate on may have compliance requirements that affect what you can do via the extension. Examples:

- GDPR: PII in flow metadata that you screenshot or back up via the extension is your responsibility to handle
- HIPAA: read access to PHI fields is mediated by your Salesforce profile; the extension does not bypass FLS
- SOX: production changes via Apply / Cross-org deploy should follow your change management process

The extension is a client tool. Your existing compliance posture for Salesforce client tools applies.

## What to do if you find a security issue

File at `github.com/ialameh/flow-id-reveal/issues` with the `security` label, or contact maintainer directly. Do not include sensitive Salesforce data in the bug report.

## References

- [Chrome extension security overview](https://developer.chrome.com/docs/extensions/develop/concepts/content-security-policy)
- [chrome.cookies API security model](https://developer.chrome.com/docs/extensions/reference/api/cookies)
- [Salesforce REST API authentication](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/intro_understanding_authentication.htm)
- [OWASP Top 10](https://owasp.org/Top10/)
