# Cookbook 04: Pre-deploy validate-only dry-run

Risk-free check before kicking off any deploy. 5 minutes.

## Scenario

You have a patched Draft in source. You want to know whether it will deploy to target cleanly. You don't want to actually deploy yet (the rollback window after a real deploy is small).

## Step 1. Open source's Draft in Flow Builder

Make sure you are on the Draft version, not the Active version. URL: `?flowId=<draft-flow-id>`.

If you accidentally scan the Active version, the Pre-deploy validateOnly will still run, but you'll be validating the wrong version against target.

## Step 2. Scan + Backup

Side panel → Scan → confirm flow loaded.

Side panel → Export → **Backup current flow metadata to clipboard** → save to file.

The backup is a safety net even though Pre-deploy is read-only on target. If you decide to run Cross-org deploy as a real write later, the backup lets you restore source if needed.

## Step 3. Open target tab

In the same Chrome window, open target SF tab. Log in.

You don't need to focus the target tab for Pre-deploy. Pre-deploy uses the CURRENTLY SCANNED tab (source). To validate against target instead, see Step 6.

## Step 4. Run Pre-deploy against source

Side panel → Audit → Pre-deploy → click **Run pre-deploy validation**.

The extension:

1. Builds an MDAPI deploy package from `state.flowMetadata`
2. POSTs to source's MDAPI SOAP endpoint with `checkOnly=true`
3. Polls deploy status every 2 seconds
4. Reports verdict

Verdicts:

- **PASSED** → flow is internally consistent in this org. Would deploy to source successfully if you ran a real deploy.
- **FAILED** → flow has issues even against its own org. Read the error.

This is useful for spotting structural problems (orphan elements, broken connectors, invalid references inside source itself).

## Step 5. Now run Pre-deploy against target

Side panel → Patch → Cross-org deploy → **Refresh target tab list** → tick the target tab.

Tick **Validate only** (it should be ticked by default).

Click **Deploy patched metadata to selected target**.

The extension runs `checkOnly=true` against TARGET org with the in-memory `state.flowMetadata`. This includes any Repoint swaps that are pending (even if not Applied).

Verdict tells you if the patched-to-target metadata would deploy cleanly to target. Common failures:

- Form not found in target (CMS publish needed)
- Form-binding collision (target has existing Draft)
- Custom field missing in target (deploy field first)
- Hardcoded Id 404 (Repoint not done for some Id)
- API version mismatch (target on older release)

## Step 6. Iterate

For each FAILED reason, fix it:

- Compare → identify which records are still MISSING in target → Clone them
- CMS Form → publish form in target, patch reference
- Compare → Draft collision check → handle existing Draft in target
- Repoint table → ensure all Ids have target values

Re-run Pre-deploy after each fix. The cycle is fast: 30 seconds per validation.

## Step 7. When Pre-deploy passes

You have high confidence the real deploy will succeed. Options:

- Untick Validate only and run again to do the real deploy in-extension
- Or push via Gearset, Change Set, sfdx with your normal pipeline (the extension's validation result transfers)

## What Pre-deploy catches vs misses

Catches:

- Schema mismatches (target lacks field, object)
- Reference resolution failures (Id rot, form not found)
- Form-binding collisions
- API version conflicts
- Most structural deploy blockers

Misses:

- Runtime errors (the validator runs static checks, not actual flow execution)
- Performance issues (governor limit risk)
- Logic errors (the flow's behaviour is correct as XML but wrong as design)
- Permission issues on the deploying user (some FLS edge cases)

For runtime + logic + perf, run the flow in a non-prod environment after deploy.

## Time

- First Pre-deploy: 2-3 min (network roundtrip + 30 second async poll)
- Subsequent iterations during fix cycle: 30-60 sec each
- Total for a typical multi-fix cycle: 10-15 min

Much faster than running real deploys to discover errors one at a time.

## Common errors and resolution time

| Error | Time to resolve |
|---|---|
| Form not found | 5-10 min (CMS publish in target) |
| Form-binding collision | 30 sec (Lifecycle button in target tab) |
| Missing custom field | Variable (depends on whether field is in another package) |
| Hardcoded Id 404 | 1-3 min (Compare → Clone or Use this Id) |
| API version mismatch | 30 sec (bump version override) |

Pre-deploy turns "deploy fails opaquely after 5 minutes" into "validate-only fails clearly in 30 seconds". Worth the time.
