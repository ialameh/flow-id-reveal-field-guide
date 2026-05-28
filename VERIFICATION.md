# Verification

What was verified, when, against which Salesforce / Chrome versions, what was confirmed, what was corrected, what remains uncertain.

## Verification scope

This guide was verified against:

- **Salesforce API version:** v62.0 (Winter '26) - minimum tested
- **Salesforce sandbox tested:** Marketing Cloud Growth + Data Cloud + CMS enabled
- **Chrome version:** 148.x (Stable channel)
- **Extension version at verification:** v0.5.2
- **Verification date:** 2026-05-28

## Verified

### Foundation chapters

- [x] **01 Mental model** - cross-org Id rot confirmed reproducible on at least one form-triggered flow (see case study 01). Pod codes verified to differ between source SilverChef pod `WG` and target sandbox pod `WF`.
- [x] **02 Architecture** - every component file confirmed present in source tree at the documented paths. Message routing flow traced end-to-end during Playwright testing.
- [x] **03 Session model** - INVALID_SESSION_ID error reproduced when fetching from lightning.force.com origin without proper auth (live test 2026-05-28). Background script `chrome.cookies.get` confirmed to return HttpOnly sid from my.salesforce.com domain. Cross-host fetch from background confirmed to bypass CORS (live test).
- [x] **04 API version negotiation** - `/services/data/` endpoint confirmed to return array of supported versions. Auto-detect picks highest. Override field accepts `vNN.N` format. Real-world failure reproduced (createConsent on v60 endpoint → 400 error) and resolved by bumping to v62.

### Operations chapters

- [x] **06 Scan tab** - DOM scan + Tooling API scan + merge confirmed via Playwright on General Enquiry AU Form Handler v1 Flow. All 3 Marketing Cloud Growth Ids returned with correct JSON paths.
- [x] **07 Compare tab** - Per-Id check confirmed via Playwright. Resolve names confirmed to return Marketing / Marketing Emails / Email for the 3 source Ids.
- [x] **08 Patch tab** - Apply patch confirmed to PATCH the Draft via Tooling API at v62+. Earlier failure at v60 (ACTIONCALL_NOT_ACTIVE_FOR_API_VERSION) reproduced and resolved.
- [x] **09 Inspect tab** - Source viewer confirmed to render `state.flowMetadata`. CMS Form helper parsing of the form reference verified against the live `marketing--Default_Content_Workspace.sfdc_cms__formHandler--MCP4JHBDDOEBHHJKMUYCEKF4YHGY` reference.
- [ ] **10 Audit tab** - Pre-deploy, Perf scan, Trigger order, Inventory verified by code review only; not yet end-to-end tested on live data.
- [x] **11 Export tab** - JSON / CSV / Backup confirmed via Playwright clipboard interaction.

### Design chapters

- [x] **12 Cross-org deploy workflow** - Change Set version-picker limitation confirmed via Salesforce Setup UI (modern Change Set bundles Active only). MDAPI deploy SOAP envelope structure verified against Salesforce documentation.
- [x] **13 Form-bound flow constraints** - "Form already associated with a draft flow version" error reproduced on attempt to create second Draft. "To publish the form handler, activate its related flow" error reproduced on Deactivate attempt while form was published.
- [x] **14 Patch and Repoint safety** - poisoned-Draft semantics confirmed by examination of patched JSON after Apply. INVERSE prefill behaviour verified via UI testing.
- [x] **15 CMS Form helper** - form reference parsing verified against live binding. Workspace + content type + content key extraction matches Salesforce CMS structure.
- [ ] **16 Pre-deploy and ZIP** - ZIP writer verified to produce MDAPI-acceptable archives (tested in Audit Pre-deploy live test). SOAP envelope verified against Metadata API documentation.
- [ ] **17 Auto-pin to CMT** - Suggestion-only feature; CMT creation path not yet implemented or verified. v0.5 candidate.

### Reference chapters

- [x] **18 Troubleshooting** - Each error documented was either personally encountered during development or reproduced during verification.
- [x] **19 Anti-patterns** - Each anti-pattern documented was either personally observed in real deploys or directly inferable from the API behavior.
- [ ] **20 Performance** - Timing references are estimated from observed development testing on single-machine setup; not benchmarked against multiple orgs / load conditions.
- [x] **21 Privacy and security posture** - Manifest permissions verified against documented usage. Network traffic verified via DevTools to only hit Salesforce host_permissions domains.
- [x] **22 FAQ** - Each answer reflects current v0.5.2 behavior or documented v0.6 candidates.

## Corrected during verification

- Initial Mental Model chapter had API version pinned to v60. Corrected when the createConsent v61+ requirement surfaced during live testing.
- Patch tab documentation initially claimed PATCH always works on Draft; corrected after Salesforce silent-drop quirk was observed on form-bound flows.
- Architecture chapter initially described `chrome.cookies.get` as readable from content scripts; corrected - only background can access cookies API.

## Uncertain or unverified

- **Behavior on Salesforce Spring '26 (v63) and Summer '26 (v64).** Documentation assumes API contract continuity. Likely-true but not actively verified.
- **Behavior on Firefox / Edge / other Chromium browsers.** Not tested.
- **Multi-flow MDAPI deploy via the extension.** Single-flow only currently; multi-flow ZIP packaging is not tested.
- **Org-pair maps storage limits.** Documented at 10MB Chrome default but not stress-tested with thousands of maps.
- **CMS Form helper auto-locate regex on flow references that use unusual workspace name characters** (e.g. spaces, special characters). May require the "replace whole reference" fallback prompt.
- **Cross-org deploy from one production tenant to another production tenant.** Tested against sandbox-to-sandbox only.

## How this guide is kept current

- Extension source repo: `github.com/ialameh/flow-id-reveal`
- Docs repo: `github.com/ialameh/flow-id-reveal-field-guide`
- Each extension release (v0.x.y) corresponds to a docs version. CHANGELOG.md tracks both.
- Documentation updates accompany feature additions and bug fixes.

If you find drift between the docs and the extension's actual behavior, file an issue.

## What to verify in your own environment

When using this guide:

1. Run a Scan on a known flow with known Ids. Confirm the Scan output matches your expectation.
2. Run a Compare against a known target tab. Confirm EXISTS / MISSING verdicts match what you'd predict by manually checking each record.
3. Run a Pre-deploy validate-only against a target. Confirm a known-good flow validates clean and a known-bad flow fails with intelligible errors.
4. Run a Backup. Save the JSON. Verify the contents match the flow's metadata.

If any of these does not behave as documented, file an issue.
