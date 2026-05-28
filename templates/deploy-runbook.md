# Deploy runbook template

Copy this into your deploy ticket. Tick items as you go.

## Flow being deployed

- Name: `<flow label>`
- Source org: `<source-alias>`
- Target org: `<target-alias>`
- Deploy mechanism: `<Gearset | Change Set | Cross-org deploy | sfdx>`
- Operator: `<your name>`
- Date: `<YYYY-MM-DD>`
- Risk level: `<low | medium | high>`

## Pre-deploy checklist

- [ ] Source backup taken (Export → Flow backup → save to file)
- [ ] Source scan run, all Ids resolved to names
- [ ] Target tab open and logged in
- [ ] Compare per-Id run, EXISTS / MISSING noted per Id
- [ ] All MISSING records cloned to target OR plan to create manually documented
- [ ] CMS form (if applicable) published in target with new content key
- [ ] CMS Form helper used to patch source Draft with target content key (if applicable)
- [ ] Repoint table populated with all target Ids
- [ ] Map saved to `<source>-to-<target>` for inverse restore later
- [ ] Pre-deploy validateOnly run against target — PASSED
- [ ] Target Draft collision check — OK
- [ ] Stakeholders notified (Slack / email)

## Deploy

- [ ] Apply patch to source Draft (if using Gearset / Change Set)
  - OR
- [ ] Cross-org deploy with Validate only UNCHECKED (writes to target)
- [ ] Gearset / Change Set / sfdx deploy ran
- [ ] Deploy result: SUCCESS / FAILED (attach screenshot or output)

## Post-deploy verification

- [ ] Switched to target tab, scanned the new Draft
- [ ] Activated new Draft in target (Lifecycle → Activate)
- [ ] End-to-end test: form submission triggers flow, runs successfully
- [ ] Total Runs counter increments
- [ ] No errors in Setup → Process Builder → Setup Errors

## Source cleanup

- [ ] Repoint INVERSE prefill applied to source (restores source Ids)
  - OR
- [ ] Lifecycle → Delete staging Draft in source
- [ ] CMS Form helper patched back to source content key (if applicable)
- [ ] Map exported to team-shared location (if new mappings discovered)

## Rollback plan (if needed)

- [ ] Restore source from backup file (Export → Restore from clipboard)
- [ ] Deactivate the new Draft in target (Lifecycle → Deactivate)
- [ ] Re-Activate the previous Active version in target (Setup → Flows)
- [ ] Notify stakeholders of rollback

## Sign-off

- Source state confirmed: ✓
- Target state confirmed: ✓
- End-to-end test passed: ✓
- Documentation updated: ✓
- Ticket / runbook closed: ✓

Operator signature: `____________________`  Date: `____________________`
