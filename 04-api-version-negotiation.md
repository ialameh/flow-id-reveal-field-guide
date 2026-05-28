# 04. API version negotiation

Salesforce releases three times a year. Each release introduces new API versions, new flow elements, and new validator rules. The extension defaults to a recent version but supports per-org auto-detection and user override.

## Why version matters

Salesforce Flow validation is API-version-sensitive in two directions.

### Endpoint version

The REST + Tooling endpoints take a version segment in the URL: `/services/data/v62.0/sobjects/...`. Each version has slightly different behaviour. A v60 endpoint may reject a flow operation that v62 accepts.

The real-world example that motivated dynamic version negotiation: the `createConsent` invocable action was introduced in API v61. A flow that uses it cannot be saved via a v60 Tooling PATCH. Salesforce returns:

```json
{
  "message": "Email_Consent (Action) - We couldn't deploy the flow because the Create Consent invocable action is unsupported in API version 60. Use API version 61 and later.",
  "errorCode": "FLOW_EXCEPTION",
  "extendedErrorDetails": [{
    "extendedErrorCode": "ACTIONCALL_NOT_ACTIVE_FOR_API_VERSION"
  }]
}
```

A pinned-v60 extension hits this. A version-aware extension picks up v61+ automatically.

### Flow's declared apiVersion

The flow XML itself declares its own `apiVersion` field. Salesforce validates the flow's elements against this declared version. The extension does not change the flow's declared version, only the version used for the endpoint URL.

## The negotiation order

The extension's `getApiVersion(apiHost)` function in `background.js` resolves in this priority:

1. **User override** in `chrome.storage.local` under key `fir.apiVersion`. If set, used unconditionally.
2. **In-memory cache** per host. If detection ran before for this host, use the cached result.
3. **Live discovery** via `GET /services/data/`. Salesforce returns an array of supported versions. Pick the highest.
4. **Hardcoded fallback** `v62.0`. Used only if all of the above fail.

## How auto-detect works

The `/services/data/` endpoint requires no authentication. It returns the catalog of API versions the org supports:

```json
[
  {"label": "Winter '26", "url": "/services/data/v62.0", "version": "62.0"},
  {"label": "Spring '26", "url": "/services/data/v63.0", "version": "63.0"},
  {"label": "Summer '26", "url": "/services/data/v64.0", "version": "64.0"}
]
```

The extension picks the highest number and caches it in memory for the session. Cache survives until the background service worker is unloaded (Chrome unloads idle service workers aggressively, so the cache is essentially per-burst-of-activity).

## User override

The side panel header has an input field labelled "API ver:". Three states:

- **Empty** (placeholder shows the auto-detected version): auto mode is active
- **Filled** (e.g. `v62.0`): override is active, all calls use this version
- **Auto button**: clears any override, re-runs detection, reports the new value
- **Clear button**: removes the override (back to auto mode)

Use override when:

- You want to reproduce a bug that only happens on a specific version
- The auto-detected version is too new and a flow element is not yet supported in your environment
- You are testing the extension's behaviour on legacy API surfaces

## Where the version is used

Every API call through background uses `getApiVersion(apiHost)`:

```js
async function toolingQuery({ apiHost, sid, soql }) {
  return sfFetch({
    apiHost, sid,
    path: `/services/data/${await getApiVersion(apiHost)}/tooling/query?q=` + encodeURIComponent(soql)
  });
}
```

The same version powers the MDAPI SOAP endpoint:

```js
const r = await fetch(`https://${apiHost}/services/Soap/m/${(await getApiVersion(apiHost)).slice(1)}`, { ... });
```

(`.slice(1)` strips the leading `v` because the SOAP endpoint takes `62.0`, not `v62.0`.)

The package.xml the extension builds for MDAPI deploys also uses this:

```xml
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
  <types><members>...</members><name>Flow</name></types>
  <version>62.0</version>
</Package>
```

The side panel reads the active version from background before building the package and substitutes the literal at template-build time.

## What if my override is invalid

The extension validates format only: `/^v\d+\.\d+$/`. A malformed override is rejected with an alert. A valid-format but unsupported-by-the-org version is sent to Salesforce, which returns a clear error (HTTP 404 on the URL or similar). The extension surfaces the error verbatim.

## Versioning gotchas

Three things to know.

### 1. Sandboxes lag prod

A sandbox refreshed before a release may report an older max version. If you scan a sandbox the day after refresh, auto-detect might return v60 while prod is on v62. The extension uses what the sandbox advertises.

### 2. Different orgs may report different versions

Source and target orgs can be on different release versions. The extension caches per-host, so each org gets its own negotiated version.

### 3. Mid-deploy version changes are awkward

If you start a pre-deploy validateOnly against a v62 endpoint and then bump the override to v63 mid-poll, the status poll may use a different version. Avoid this by not changing version during deploys.

## Programmatic detect from the side panel

You can run detect manually:

```
Side panel header → click "Auto" button
```

The background clears the cache for the active SF tab's host, removes any override, and re-runs detect. An alert shows the result. The page reloads the field.

Equivalent JS in the side panel context:

```js
chrome.runtime.sendMessage({
  kind: "api-version-detect",
  originHost: location.hostname.replace("chrome-extension://", "")  // or use scan's origin
}, resp => console.log(resp));
```

## References

- [Salesforce API release notes](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_api_release_notes.htm)
- [REST API version catalogue endpoint](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/dome_versions.htm)
- [Flow runtime API versions](https://help.salesforce.com/s/articleView?id=sf.flow_distribute_api_version.htm)
