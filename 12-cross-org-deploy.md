# 12. Cross-org deploy workflow

Three ways to ship a flow from source to target. Each handles Id rot differently. Pick by org topology + tooling availability.

## The three deploy mechanisms

### Gearset (or similar third-party DevOps tools)

Paid tool. Pulls source metadata via its own retrieve, optionally diff-shows what changes, pushes to target via MDAPI. Version-aware: can ship a specific flow version including Draft.

Strengths:

- Cross-tenant (any org pair, no Salesforce-side connection required)
- Automated, repeatable, schedulable
- Diff visualization

Weaknesses:

- Cost
- Same Id rot vulnerability — moves the bytes as-is. Patch the source flow first via the extension, then let Gearset ship.

### Change Sets (Salesforce-native)

Free. Outbound Change Set from source, Inbound Change Set in target. Requires a Deployment Connection between source and target (sibling sandboxes off same prod, or sandbox↔prod).

Strengths:

- Free and built-in
- Salesforce-supported, predictable

Weaknesses:

- Limited org topology (no cross-tenant)
- Cannot pick Draft versions (modern Change Set UI bundles Active version only)
- Slow (manual upload + deploy click)
- Same Id rot vulnerability

To ship a Draft via Change Set, you need to activate the Draft in source first (intrusive — see Chapter 13 form-binding constraints), or use a different deploy path.

### sfdx CLI / Salesforce CLI

Free. CLI-driven. Can specify exact version via sourcepath or manifest.

```bash
sf project retrieve start --metadata Flow:<MyFlow>
sf project deploy start --source-dir force-app/main/default/flows
```

Strengths:

- Scriptable, CI-friendly
- Version-aware
- Free

Weaknesses:

- Setup overhead (CLI install, project structure, sfdx-project.json)
- Same Id rot vulnerability
- No GUI for non-CLI users

### Cross-org MDAPI deploy from the extension

Free. In-extension. Takes the source's flow metadata (in-memory, with any Repoint swaps applied), packages it inline, ships via MDAPI SOAP to a target tab's org.

Strengths:

- One-flow surgical deploys
- Patch + deploy in same session
- Validate-only dry-run built in
- Cross-tenant capable (no SF-side connection needed)
- Free

Weaknesses:

- One flow at a time (no multi-component bundles)
- Relies on the extension's heuristic JSON-to-XML serializer (not authoritative)
- No diff visualization
- Browser-bound (no CI integration)

See [Chapter 16](16-predeploy-and-zip.md) for the MDAPI + ZIP details.

## Decision matrix

| Scenario | Recommended path |
|---|---|
| Same-prod sandbox to sibling sandbox, no Drafts needed | Change Set |
| Sandbox → prod, established pipeline | Gearset or sfdx in CI |
| Sandbox → prod, ad-hoc emergency hotfix | Extension Cross-org deploy |
| Different prod tenants | Gearset, sfdx, or Extension |
| Need to ship a specific Draft version | Gearset, sfdx, or Extension (not Change Set) |
| Quick iteration during development | Extension (validate-only + write) |
| Auditable, repeatable production change | Gearset or sfdx in CI |

## The cross-org patch-and-ship workflow

End-to-end pattern that works with any deploy tool.

```
1. Source org: open the flow (V_n Active or V_(n+1) Draft) in Flow Builder
2. Extension → Scan → identify hardcoded Ids
3. Extension → Compare → target tab → identify which Ids need fixing
4. For each MISSING Id:
   - Extension → Compare → Clone → target (if data record)
   - OR manually create in target's CMS / Setup (if metadata or CMS content)
5. For each EXISTS Id:
   - Extension → Compare → Use this Id (autofills Repoint)
6. Extension → Patch → Repoint → set map name → Save swaps to map → Apply patch
   (This writes target Ids INTO source Draft. Source Draft is now POISONED for source.)
7. Compare → Draft collision check → target must have no Draft for this flow
8. If target has Draft: switch to target tab in extension → Patch → Lifecycle → Activate or Delete
9. Deploy via your chosen tool (Gearset, Change Set, sfdx, or Extension Cross-org deploy)
10. In target: extension → Patch → Lifecycle → Activate the new Draft
11. In source: extension → Patch → Repoint → Prefill INVERSE → Apply
    (Restores source Ids to source Draft. Source is back to normal.)
    OR: extension → Patch → Lifecycle → Delete this Draft version
```

The two safety bookends (Backup before Apply, Restore after deploy) prevent leaving source in a broken state.

## When the extension's Cross-org deploy IS the right tool

Use it when:

- You are iterating fast and don't want to push to Git + run CI for every test
- You need a one-flow deploy without scope creep
- You want to validate-only against multiple target orgs in sequence
- You don't have Gearset and Change Set won't pick the Draft
- Source is a personal sandbox and you don't want it in Git

The validate-only path is also useful as a pre-flight check before kicking off a Gearset or sfdx deploy. Catches issues fast.

## When the extension is NOT the right tool

Use Gearset / sfdx / Change Set when:

- You are shipping multiple components (Apex classes, custom objects, profiles, etc.) along with the flow
- You need CI integration
- You need a paper trail for audit
- You are shipping to production (extension's heuristic XML serializer is fine for sandboxes but use authoritative tooling for prod)

## What changes when targets are different tenants

If source and target are on different production tenants (different `00D...` org IDs), Change Set is unavailable. Gearset works. sfdx works. Extension Cross-org deploy works.

The CMS content key handling becomes more relevant on different tenants because CMS workspaces likely have different content. Plan CMS form publish ahead of time.

## What changes when target is production

- Pre-deploy validate-only is mandatory. Always.
- Manual production deploys via the extension are possible but not recommended.
- Use Gearset or sfdx with proper change management.
- The extension still adds value: scan + patch + validate before kicking off the production deploy.

## References

- [Salesforce Change Sets overview](https://help.salesforce.com/s/articleView?id=sf.changesets.htm)
- [Salesforce CLI deploy documentation](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference_project_commands_unified.htm)
- [Gearset deployment documentation](https://gearset.com/docs)
- [Salesforce DevOps Center](https://help.salesforce.com/s/articleView?id=sf.devops_center_overview.htm)
