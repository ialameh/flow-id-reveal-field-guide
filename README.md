# Flow ID Reveal Field Guide

A practical handbook for cross-org Salesforce Flow Builder Id management with the Flow ID Reveal Chrome extension. Aimed at architects, dev leads, and admins who deploy form-triggered flows, Data Cloud flows, and Marketing Cloud Growth flows across multiple orgs and hit the cross-org Id rot problem.

This guide covers the problem, the extension, the cross-org deploy workflow, the form-bound flow constraints, and the operational patterns that keep flows shippable across sandboxes and production orgs.

## Who this is for

- Salesforce architects responsible for deploy pipelines that include flows
- Dev leads troubleshooting pipeline failures with messages like "An unexpected error occurred. Please include this ErrorId if you contact support"
- Admins authoring CMS form-triggered flows in source orgs and shipping them to UAT and prod
- Anyone who has stared at a `0XlWG0000000Ihh0AE` in a flow XML and asked "why does this Id break in target"

## The problem in one paragraph

Flow Builder dropdowns store the underlying Salesforce record Id when you pick a Communication Subscription, an Engagement Channel Type, a CMS form binding. Those Ids are pod-specific. When the flow ships from one org to another, the target org has different Ids for the same logical records, so deploy fails with a generic error and no line number. The extension surfaces every hardcoded Id inside the flow, resolves it to a name, compares it against any other open org tab, clones missing records into target, patches the flow metadata, and optionally deploys the patched flow cross-org via Metadata API. The end result: a deploy that lands.

## What the extension does

Six tabs in the side panel.

| Tab | Purpose |
|---|---|
| **Scan** | Find every Salesforce Id in the active Flow Builder page (DOM scan + Tooling API metadata scan). Resolve to names. |
| **Compare** | Cross-org per-Id check, full JSON diff between same flow in two orgs, version diff between flow versions, Draft collision check. |
| **Patch** | Repoint hardcoded Ids, save org-pair Id mappings, generic search/replace, cross-org MDAPI deploy, lifecycle (Activate/Deactivate/Delete), auto-pin to Custom Metadata suggestion. |
| **Inspect** | Raw flow JSON source viewer with search, dependencies analyzer, picklist literal extractor, field usage map, CMS Form helper. |
| **Audit** | Pre-deploy MDAPI validateOnly dry-run, perf scan (DML/SOQL in loops), trigger order analyzer, flow inventory dashboard, reverse Id lookup across all flows. |
| **Export** | Scan results as JSON/CSV, raw flow metadata viewer, backup/restore of current flow metadata. |

## Chapter map

### Quickstart

- [00. Quickstart](00-quickstart.md) - install, focus a SF tab, scan, resolve

### Foundations

- [01. Mental model](01-mental-model.md) - the cross-org Id rot problem, why it happens, how the extension addresses it
- [02. Architecture](02-architecture.md) - Manifest V3, background service worker, content script, side panel UI, message routing
- [03. Session model](03-session-model.md) - sid cookies, HttpOnly, `*.my.salesforce.com` vs `*.lightning.force.com`, CORS implications
- [04. API version negotiation](04-api-version-negotiation.md) - per-org auto-detect via `/services/data/`, user override path, per-flow API version awareness
- [05. Salesforce Id taxonomy](05-id-taxonomy.md) - prefix table, 15 vs 18 char Ids, classifier rules, why some Ids are not data records (CMS content keys)

### Operations

- [06. Scan tab](06-scan-tab.md)
- [07. Compare tab](07-compare-tab.md)
- [08. Patch tab](08-patch-tab.md)
- [09. Inspect tab](09-inspect-tab.md)
- [10. Audit tab](10-audit-tab.md)
- [11. Export tab](11-export-tab.md)

### Design and Engineering

- [12. Cross-org deploy workflow](12-cross-org-deploy.md) - Gearset vs Change Set vs MDAPI direct deploy
- [13. Form-bound flow constraints](13-form-bound-flow-constraints.md) - CMS form coupling, Draft uniqueness, "form already associated" errors
- [14. Patch and Repoint safety](14-patch-and-repoint-safety.md) - poisoned Draft semantics, save-swap-to-map, INVERSE restore
- [15. CMS Form helper deep dive](15-cms-form-helper.md) - workspace, content type, content key, cross-org GUID handling
- [16. Pre-deploy dry-run and inline ZIP builder](16-predeploy-and-zip.md) - MDAPI validateOnly, SOAP envelope, JSON-to-XML round-trip
- [17. Auto-pin to Custom Metadata](17-auto-pin-to-cmt.md) - extracting hardcoded literals into environment-specific records

### Reference

- [18. Troubleshooting](18-troubleshooting.md) - every error you might hit, with cause + fix
- [19. Anti-patterns](19-anti-patterns.md) - what not to do and why
- [20. Performance tips](20-performance.md) - keeping scans fast on big flows and big orgs
- [21. Privacy and security posture](21-privacy-and-security.md) - what data the extension reads, where it goes
- [22. FAQ](22-faq.md)

### Worked material

- [Cookbook](cookbook/README.md) - four end-to-end worked examples
- [Templates](templates/README.md) - copy-paste skeletons
- [Case studies](case-studies/README.md) - anonymised real incidents

### Glossary

- [23. Glossary](23-glossary.md)

## 30-second version

Five rules of thumb.

1. Scan first. Never assume a flow has zero hardcoded Ids until you see the scan output.
2. Compare against the target tab before you ship. Missing or different Ids in target predict deploy failure with certainty.
3. Patch into a Draft you can safely poison. Never activate a patched-with-target-Ids Draft in source.
4. Save the swap into the Org-pair maps. The inverse restore is one click instead of an archeology project.
5. For form-triggered flows, the CMS content key is a separate Id rot dimension. Patch it via the CMS Form subpanel after publishing the form in target's CMS.

## Project status

v0.5.2 as of 2026-05-28. Six tabs functional. Tested against Salesforce sandboxes with Marketing Cloud Growth + Data Cloud + CMS enabled.

## License

CC BY 4.0. See [LICENSE](LICENSE).

## Tone

Plain conversational prose. No marketing. Cited.
