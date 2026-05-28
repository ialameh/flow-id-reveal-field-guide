# 03. Session model

The hardest two-day debug in building this extension was figuring out how to authenticate API calls from a browser extension. This chapter is the saved version of that knowledge.

## Salesforce uses two hosts per org

When you log into Salesforce Lightning Experience, your browser ends up with cookies on two related hosts:

| Host | Used by | Cookie scope |
|---|---|---|
| `<domain>.my.salesforce.com` | REST + Tooling + SOAP API endpoints | `sid` cookie, HttpOnly, Secure |
| `<domain>.lightning.force.com` | Lightning Experience frontend (the UI) | `sid` cookie (different value), HttpOnly, Secure |

The two `sid` values are unrelated. The Lightning one authenticates the LEX shell. The my.salesforce.com one authenticates API calls. Both are issued by the OAuth flow at login time.

## Why this matters for the extension

A naive first version of the extension tried to fetch the REST API from the Lightning page directly:

```js
// From a *.lightning.force.com page:
fetch('/services/data/v60.0/sobjects/CommSubscription/0XlWG0000000Ihh0AE')
```

This fails. Two failure modes overlap:

### Failure 1. The Lightning sid is rejected by the REST endpoint

Lightning's sid is scoped to the LEX UI. Hitting `/services/data/...` on `*.lightning.force.com` returns:

```json
[{"errorCode":"INVALID_SESSION_ID","message":"Session expired or invalid"}]
```

Status 401. The REST endpoint exists at the lightning.force.com URL (Salesforce serves it from both hosts), but the validator requires the API-scoped sid.

### Failure 2. Cross-host fetch is blocked by CORS

A correct workaround would be to fetch directly from my.salesforce.com:

```js
fetch('https://<domain>.my.salesforce.com/services/data/v60.0/sobjects/...', {
  credentials: 'include'
})
```

But the browser's CORS preflight rejects this: the request originates from `https://<domain>.lightning.force.com`, the target is `https://<domain>.my.salesforce.com`, and Salesforce's response does not include the cross-subdomain `Access-Control-Allow-Origin` header. Browser logs `TypeError: Failed to fetch`.

## The extension's solution

Use the background service worker.

Background scripts run in the extension origin (`chrome-extension://<id>`). The manifest declares `host_permissions` for `*.my.salesforce.com`, which exempts background fetches from CORS rejection for that host. Background scripts also have the `cookies` permission, which lets them read HttpOnly cookies via `chrome.cookies.get()`.

The flow:

```js
// In background.js
async function readSidCookie(apiHost) {
  return new Promise(resolve => {
    chrome.cookies.get({ url: "https://" + apiHost + "/", name: "sid" }, c => {
      resolve(c && c.value ? c.value : null);
    });
  });
}

async function sfFetch({ apiHost, sid, path }) {
  return fetch("https://" + apiHost + path, {
    headers: { "Authorization": "Bearer " + sid, "Accept": "application/json" }
  });
}
```

`Authorization: Bearer <sid>` is the standard way to pass an OAuth-issued sid to the Salesforce REST API. Equivalent to the `?access_token=` query parameter but cleaner.

## Host derivation

When the side panel asks the background to do something, it passes `originHost` (the host of whatever SF tab is being scanned). The host is one of:

- `<domain>.lightning.force.com`
- `<domain>.my.salesforce.com`

Background derives the API host with a simple substitution:

```js
function deriveApiHost(urlOrHost) {
  const host = (urlOrHost.startsWith("http"))
    ? new URL(urlOrHost).hostname
    : urlOrHost;
  return host.replace(".lightning.force.com", ".my.salesforce.com");
}
```

Background then reads the sid cookie for the my.salesforce.com derivation, regardless of which host the side panel passed in.

## What about sandboxes and CMS Experiences?

The same pattern applies. Sandbox domains follow the `<base>--<sandbox>.sandbox.my.salesforce.com` format. CMS Experience Cloud sites live on `<domain>.force.com` or `<domain>.my.site.com`. The manifest's `host_permissions` covers them all.

For Experience sites, the sid cookie is again on `*.my.salesforce.com` (the orgs's API host), not on the Experience host. The extension does not directly hit Experience site URLs.

## Sandbox refresh and sid

When you refresh a sandbox, all existing sessions invalidate. The extension does not detect this. After a refresh:

- Existing tabs show stale sids. Hitting any API call returns INVALID_SESSION_ID.
- Hard-refresh the SF tab to re-auth. Salesforce will SSO you back in.

The extension's scan will surface INVALID_SESSION_ID as the reason on the error rows. Refresh the tab and rescan.

## What we explicitly do NOT do

- **We do not store the sid anywhere.** It is read fresh from cookies on every call. If you uninstall the extension, no Salesforce credentials persist.
- **We do not refresh sid tokens.** OAuth refresh tokens are not exposed in the cookie. We rely on the browser's session being current. Logged-out users see auth errors and re-log in via the SF login page.
- **We do not log sid values.** No console.log, no analytics, no debug dump includes the sid. The cookie value never leaves the extension.

See [Chapter 21](21-privacy-and-security.md) for the full privacy posture.

## What the user has to do

Stay logged into both source and target Salesforce orgs in the same browser. The extension uses `chrome.cookies.get()` per host, so each org needs its own active session in your browser.

If you use Chrome profiles to isolate orgs (one Chrome profile per client), install the extension in each profile. Each profile has its own cookie jar.

## A test you can run

In the side panel, focus a SF tab. Then open Chrome DevTools on the SF tab and run:

```js
fetch('/services/data/').then(r => r.json()).then(console.log)
```

This is the unauthenticated catalog endpoint. It returns the API version list. If you see a populated array, the domain is reachable and Salesforce will accept your sid on subsequent authenticated calls. The extension's API-version-detect feature uses this endpoint.

For an authenticated round-trip, run from the SF tab console:

```js
const flowId = new URL(location.href).searchParams.get('flowId');
const myHost = location.hostname.replace('.lightning.force.com', '.my.salesforce.com');
// Note: this WILL fail with CORS from the lightning page. Run from the extension side panel
// console instead, where chrome.runtime.sendMessage proxies through background.
```

The extension uses this exact path internally. It works because the background script bypasses both the wrong-sid and CORS issues.

## References

- [Salesforce REST API session authentication](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/intro_understanding_authentication.htm)
- [chrome.cookies HttpOnly access](https://developer.chrome.com/docs/extensions/reference/api/cookies)
- [CORS for cross-subdomain](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [Salesforce sandbox refresh impact on sessions](https://help.salesforce.com/s/articleView?id=sf.data_sandbox_refresh.htm)
