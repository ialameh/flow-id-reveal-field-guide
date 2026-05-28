# 16. Pre-deploy dry-run and inline ZIP builder

The Audit → Pre-deploy and Patch → Cross-org deploy tabs both build an MDAPI deploy package in the browser and POST it to Salesforce's SOAP endpoint. This chapter covers how that works.

## The deploy package format

A Salesforce MDAPI deploy package is a ZIP file containing:

```
package.xml           # manifest of what's being deployed
flows/               # one file per Flow component
  <FullName>.flow-meta.xml
```

The extension builds this in-memory with three pieces of code:

1. **JSON → XML serializer** (`metadataJsonToFlowXml`) - converts the Tooling-API JSON metadata back to MDAPI XML
2. **Minimal ZIP writer** (`buildSimpleZip`) - RFC 1952 / PKZIP STORE mode, no compression
3. **Base64 encoder** (`arrayBufferToBase64`) - chunked encode for large buffers

The resulting base64 string is the `<ZipFile>` value in the SOAP `deploy()` envelope.

## JSON → XML serializer

The Tooling API returns flow metadata as JSON. MDAPI deploy expects `.flow-meta.xml`. The extension's serializer:

```js
function metadataJsonToFlowXml(md) {
  function emit(name, value, indent = 1) {
    if (value == null) return "";
    if (Array.isArray(value)) return value.map(v => emit(name, v, indent)).join("");
    if (typeof value === "object") {
      const inner = Object.keys(value).map(k => emit(k, value[k], indent + 1)).join("");
      return `${pad}<${name}>\n${inner}${pad}</${name}>\n`;
    }
    return `${pad}<${name}>${xmlEscape(value)}</${name}>\n`;
  }
  const body = Object.keys(md).map(k => emit(k, md[k])).join("");
  return `<?xml version="1.0" encoding="UTF-8"?>\n<Flow xmlns="...">\n${body}</Flow>\n`;
}
```

Each JSON object becomes an XML element. Arrays become repeated elements. Primitives become text content with XML-escaping.

### Heuristic, not authoritative

This serializer is best-effort. Salesforce's official MDAPI retrieve produces canonical `.flow-meta.xml`. The extension's serializer may produce slightly different output for:

- Field ordering (JSON object keys are unordered; XML element order matters for some validators)
- Empty elements (the serializer skips null; Salesforce may emit empty tags)
- Boolean serialization (the extension emits `<x>true</x>`; Salesforce may emit `<x/>` for true)
- Numeric precision (rare edge case)

For most flows the serializer produces deploy-acceptable output. For complex edge cases, retrieve via sfdx and use the canonical XML.

## Minimal ZIP writer

The extension implements a STORE-mode ZIP (no compression) in ~100 lines. STORE is the simplest valid ZIP format and is acceptable to MDAPI.

Structure per file:

```
Local File Header (30 bytes + filename)
File data (raw bytes)
```

Plus per file:

```
Central Directory Record (46 bytes + filename)
```

Plus once at end:

```
End of Central Directory record (22 bytes)
```

The CRC-32 is computed via a precomputed table. Total dependencies: zero (pure JS, browser TextEncoder).

Why STORE instead of DEFLATE: smaller code, no need for pako or browser CompressionStream. Tradeoff: package size is larger by 30-70%. Flow metadata is small enough that the size impact is negligible (typically <50KB).

## Base64 encoder

Large ArrayBuffers cannot be `String.fromCharCode.apply(null, ...)` in one call (V8 throws on >100k args). The extension chunks:

```js
function arrayBufferToBase64(buf) {
  const bytes = new Uint8Array(buf);
  let s = "";
  const chunk = 0x8000;  // 32k chars at a time
  for (let i = 0; i < bytes.length; i += chunk) {
    s += String.fromCharCode.apply(null, bytes.subarray(i, i + chunk));
  }
  return btoa(s);
}
```

Reasonable for flow packages up to ~1MB raw (most are <100KB).

## The SOAP envelope

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
                  xmlns:met="http://soap.sforce.com/2006/04/metadata">
  <soapenv:Header>
    <met:SessionHeader><met:sessionId>${sid}</met:sessionId></met:SessionHeader>
  </soapenv:Header>
  <soapenv:Body>
    <met:deploy>
      <met:ZipFile>${zipBase64}</met:ZipFile>
      <met:DeployOptions>
        <met:checkOnly>${validateOnly ? "true" : "false"}</met:checkOnly>
        <met:rollbackOnError>true</met:rollbackOnError>
        <met:singlePackage>true</met:singlePackage>
      </met:DeployOptions>
    </met:deploy>
  </soapenv:Body>
</soapenv:Envelope>
```

POSTed to `https://<apiHost>/services/Soap/m/<version>` with `Content-Type: text/xml; charset=UTF-8` and `SOAPAction: deploy`.

The response is XML containing an `<id>` element (the AsyncResult Id) and a `<state>` element (Queued, InProgress, Completed, Failed).

## Status polling

After submit, the extension polls via `checkDeployStatus`:

```xml
<met:checkDeployStatus>
  <met:asyncProcessId>${asyncId}</met:asyncProcessId>
  <met:includeDetails>true</met:includeDetails>
</met:checkDeployStatus>
```

Polled every 2 seconds for up to 60 seconds (30 attempts). When the response contains `<done>true</done>`, polling stops. Verdict is based on `<success>true|false</success>`.

The full XML response is rendered to the output panel. Errors are visible in the response body with component name + line + error message.

## Validate-only vs write

The only difference is the `checkOnly` flag in the SOAP envelope. Everything else (package build, ZIP, base64, SOAP, polling) is identical.

For the same flow + target combination:

- Validate-only: target's flow metadata unchanged; the deploy validator runs all the same checks
- Write: target receives a new Draft (or updates existing Draft, or replaces according to package.xml semantics)

Always run validate-only first. If it passes, switch the flag and run again for the write.

## What the deploy actually does in target

For a `Flow` component in `package.xml`:

| Target state | Result |
|---|---|
| No flow with this FullName exists | Creates V1, status = Active or Draft per FlowDefinition settings |
| Flow exists with Active V_n, no Draft | Creates new Draft V_(n+1). V_n stays Active. |
| Flow exists with Active V_n + Draft V_(n+1) | Replaces V_(n+1) Draft. V_n stays Active. |
| Flow exists with only Draft V_n | Replaces V_n Draft. No Active version. |

The package.xml uses `<members>` to identify the flow by FullName (its API name), not by record Id. So the deploy is FullName-routed, which is what makes cross-org work.

## Common Pre-deploy failures + fixes

| Error | Cause | Fix |
|---|---|---|
| `INVALID_TYPE: Cannot find Flow type` | API version mismatch | Bump API version override |
| `<flow name> - field <FieldName> does not exist` | Target lacks a referenced custom field | Deploy field first |
| `<form name> - cannot find form` | CMS form not published in target | CMS Form helper → publish in target |
| `ACTIONCALL_NOT_ACTIVE_FOR_API_VERSION` | Action requires newer version than the endpoint | Bump API version (auto-detect picks highest) |
| `The form you selected is already associated with a draft flow version` | Target has competing Draft | Lifecycle → Activate or Delete on target |
| `FIELD_INTEGRITY_EXCEPTION: hardcoded Id does not exist` | The classic cross-org Id rot | Patch the Ids first via Repoint |

## Why use validate-only at all

Three reasons:

- **Free.** Doesn't write to target. Doesn't count against deploy quotas for production.
- **Fast iteration.** Catch issues in seconds, fix, retry.
- **Same validator as production deploy.** Whatever validate-only passes will pass the production deploy too (modulo race conditions on org state changes between dry-run and real run).

Pair the extension's Pre-deploy validate-only with the Compare → Draft collision check + the CMS Form check. Together they cover 90%+ of cross-org failure modes.

## References

- [Metadata API deploy() reference](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_deploy.htm)
- [Metadata API package.xml format](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/manifest_samples.htm)
- [PKZIP STORE mode (RFC 1952)](https://datatracker.ietf.org/doc/html/rfc1952)
- [Flow metadata XML format reference](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/meta_flow.htm)
