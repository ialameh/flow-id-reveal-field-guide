# 20. Performance tips

Keeping the extension fast on big flows and big orgs.

## Scan timing reference

Observed on real flows (your numbers will vary with network latency and org size):

| Flow size | Scan time | Resolve all time (per Id) |
|---|---|---|
| Small (1-5 actions) | 100-200ms | 100ms |
| Medium (10-30 actions) | 200-500ms | 100ms |
| Large (50+ actions, big apexValue) | 500ms-2s | 100ms |
| Very large (100+ elements) | 2-5s | 100ms |

The dominant scan cost is the Tooling API round-trip + JSON parsing. Local walking is microseconds.

Resolve calls are sequential. 30 Ids × 100ms = 3 seconds. Sequential is intentional (rate-limit polite). Bulk concurrent would be faster but risks 429s on aggressive APIs.

## Reverse Id lookup cost

Audit → Reverse Id lookup pulls EVERY active or draft flow's full Metadata:

```sql
SELECT Id, MasterLabel, VersionNumber, Status, Metadata
FROM Flow
WHERE Status IN ('Active', 'Draft')
```

Cost grows linearly with flow count + average flow size:

| Org | Approx flows (A+D) | Reverse lookup time |
|---|---|---|
| Small org | 20 | 5-10 seconds |
| Medium | 100 | 30-60 seconds |
| Large | 500+ | 2-5 minutes |

Mitigations:

- Use Inventory → filter by name, then click into specific flows
- Narrow the SOQL by adding `AND MasterLabel LIKE '%marketing%'` (edit the source)
- Cache results across reverse lookups (v0.6 candidate)
- Run during off-hours if you do a full lookup

## Cross-org JSON diff size limit

The diff is capped at 5000 lines per side. Larger truncate with a notice.

For very large flows (multi-thousand lines), use:

- Inspect → Source on both sides, manual visual diff with Cmd+F
- Or use `sfdx project retrieve` + a proper diff tool (VS Code, GitHub PR view)

## Storage growth

`chrome.storage.local` for the extension caps at 10MB (Chrome default for `storage.local`).

The Id maps + API version override are tiny (hundreds of bytes typically). You can store ~1000 named org-pair maps before hitting the cap.

If you exceed the cap, Chrome silently fails the write. Check via DevTools on the side panel:

```js
chrome.storage.local.getBytesInUse(null, b => console.log('Bytes used:', b))
```

10485760 is the cap.

## Service worker lifecycle

Chrome aggressively unloads idle MV3 service workers. The extension's background worker may unload between operations.

Symptoms:

- First call after a 5+ minute idle period takes ~200ms longer (cold-start the SW)
- In-memory caches (`versionCache`, etc.) are lost when the SW reloads
- No persistent state across SW reloads (use `chrome.storage` for that)

Not a bug. Designed behaviour. The extension is built to tolerate cold SW reloads — every API call resolves the API version freshly if cache is empty.

## DOM scan on large pages

Lightning Setup pages can have 10k+ DOM elements + 100+ shadow roots. The DOM scan visits every element and every open shadow root.

Cost: ~100-300ms on a typical Setup page. Up to ~1 second on extremely complex pages (Flow Builder canvas with 50+ elements).

Not configurable. The walker is already optimised (single pass per shadow tree, no re-querying).

## Network latency mitigation

The extension makes one HTTP request per API call. Round-trip is dominated by network latency, not Salesforce processing time.

Tips:

- Same-region as your sandbox helps (AU sandbox + AU Chrome = ~50ms RTT; US sandbox + AU Chrome = ~250ms RTT)
- VPN can add 50-100ms; turn off when working on Salesforce
- Sandbox-on-sandbox-pod (e.g. NA45) is fast; sandbox-on-prod-pod (e.g. NA13) varies by load

## Memory footprint

Side panel: ~20-50MB depending on state size (flow metadata can be large).
Background SW when active: ~10MB.
Content script per SF tab: ~5MB.

For 3 SF tabs open + side panel + background = ~50MB total. Negligible on modern machines.

If `state.flowMetadata` is huge (single flow with multi-MB inline data), the side panel can balloon. Mitigations:

- Clear state.flowMetadata after Apply by re-Scanning (or manually `state.flowMetadata = null`)
- Restart Chrome periodically

## Concurrent operations

The extension serializes most operations. You cannot:

- Run Compare while Scan is in progress
- Run Cross-org deploy while Apply is in progress
- Run two Resolves concurrently (the bulk Resolve is sequential by design)

You can:

- Switch tabs in the side panel while an operation runs in the background
- Open multiple Chrome windows with separate side panel instances (each is isolated)

## When the extension feels slow

Diagnose by:

1. Open DevTools on the side panel → Performance tab → record a Scan
2. Look at the flame chart for the hot path
3. Check Network tab for slow HTTP calls
4. Check Console for warnings

Common culprits:

- Tooling API > 1 second: org is under load, or you're far from the data center
- Render hang: a huge JSON diff (>5000 lines per side) triggers slow rendering
- Save spike: writing to `chrome.storage.local` during heavy use can briefly block

File performance bugs at `github.com/ialameh/flow-id-reveal/issues` with the Performance recording exported.

## References

- [Chrome MV3 service worker lifecycle](https://developer.chrome.com/docs/extensions/develop/concepts/service-workers/lifecycle)
- [chrome.storage limits](https://developer.chrome.com/docs/extensions/reference/api/storage#storage_areas)
- [Salesforce API limits per org edition](https://developer.salesforce.com/docs/atlas.en-us.salesforce_app_limits_cheatsheet.meta/salesforce_app_limits_cheatsheet/salesforce_app_limits_platform_api.htm)
