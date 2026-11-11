---
name: configure-workunit-program
description: Use when creating or editing an Interact Work Unit's conduct, phase states, GRAFCET sequences, actions, transitions or phase CI/CP in Atlas. Triggers on "configure la conduite", "crée une séquence Grafcet", "modifie cette étape", "corrige la réceptivité" or "CI/CP de phase". Route equipment CI/CP to configure-conditions and explanations to interact-guide; an explanation authorizes no mutation.
---

# Configure a Work Unit's conduct

Read the [connector and recovery procedure](../configure-plc/references/session-and-recovery.md),
[restitution](../configure-plc/references/restitution.md) and the
[conduct reference](references/conduct.md). Use those shared procedures for
session, authorization, recovery and reporting; the reference below supplies
only the conduct-specific contract. This workflow uses remote MCP and bundled
files alone. Shell, checkout, scripts, hooks, subagents, tool search, a browser
and MCP Apps are not prerequisites.

## Establish the owner and intent

Name the owner **before proposing a write**: an equipment's CI/CP belong to
[configure-conditions](../configure-conditions/SKILL.md); the Work Unit's phase
CI/CP, state diagram and sequences belong here. For an unqualified "CI/CP" or
an ambiguous repère, ask equipment or phase and which entity. Do not infer the
owner from the word "interlock". An explanation alone goes to
[interact-guide](../interact-guide/SKILL.md) and authorizes no mutation.

Check the tools and schemas actually exposed on the chosen connector:
`atlas_search`, `atlas_interact_plc_list`,
`atlas_interact_get_workunit_program` with `includeEditDetails`,
`atlas_interact_get_workunit_program_references` and
`atlas_interact_apply_workunit_program_changes`. Check
`atlas_interact_list_channels` when channel context is needed. A bundled skill
does not establish that these tools are installed. A missing tool or edit
option blocks its step and dependent writes under the shared recovery
procedure: name the tool, connector, blocked step and effects already confirmed.
Do not substitute an endpoint or change connector/environment.

## Read the current conduct and its references

1. Resolve the Work Unit with `atlas_search` and `types: ["workUnit"]`.
   In `results`, verify `type`, `id`, `siteId`, `identifier` and `target`;
   `totals` counts all matches per type. `siteId` favors that site in ranking,
   without excluding other sites. Disambiguate matches and use the exact
   `identifier` for any later `physicalId` input. Read `atlas_interact_plc_list` for that
   site, retaining the requested PLC's name, id, pin, equipment and source ids.
   Preserve the conduct's current PLC unless reassignment is explicitly wanted.
2. Call `atlas_interact_get_workunit_program` with the verified `workUnitId`
   and `includeEditDetails: true`. Retain the complete baseline, `hasProgram`,
   `plcId`, `revision`, diagnostics and typed details. Require
   `editDetailsIncluded: true`; empty detail arrays without it are not proof of
   an empty conduct. With no program, creation uses `expectedRevision: null`;
   otherwise use exactly the revision read. Never reconstruct editable values
   from `valueText`, `receptivityText` or `documentJson`.
3. Resolve the needed equipment ids with `atlas_search` and
   `types: ["equipment"]`, verifying their site and the PLC's equipment list,
   and channel ids through its source pages. Follow the shared channel
   pagination procedure. Read activity, polarity, units and ranges when the
   proposal depends on them. With `workUnitId`, the chosen `plcId` and 1–100
   `{ kind: equipment | io_channel, id }` selectors per call, use
   `atlas_interact_get_workunit_program_references` for their actual interfaces.
   Keep its `standardVersionId` consistent with the PLC read; refresh changed
   context before proposing. This read also works before a conduct exists.
   Only returned fields and their `commandable` / `actionTargetable` flags
   establish valid action targets; names or IO direction do not establish them.
4. For missing preparation, follow [configure-plc](../configure-plc/SKILL.md),
   [configure-equipment](../configure-equipment/SKILL.md),
   [link-plc-data-source](../link-plc-data-source/SKILL.md) or
   [import-register-map](../import-register-map/SKILL.md) as appropriate, within
   the request. Resume with fresh conduct, PLC and reference reads after those
   changes. Do not create a PLC, choose a new pin or modify the standard here.

## Propose and review the differences

5. Build only the requested targeted changes using the connected tool's
   schema and the [conduct operations](references/conduct.md#targeted-changes).
   Present before/after differences: owner, states/numbers, link endpoints,
   sequence owner, initial step, actions, OU priority, typed conditions and
   durations. Name retained content, deletions and their full subtree effects,
   layout effects and any PLC reassignment. Cite sources and gaps through
   restitution. A supplied functional analysis stays in the conversation.
6. Clarify an unresolved owner, operand, value, unit or sense. Preserve safety
   clauses; never drop one to make a batch acceptable. An incomplete slot or
   empty group can remain a draft only when that is the authorized result;
   name what is incomplete and the diagnostics. Review those differences and
   effects under the shared authorization procedure before applying. A changed
   proposal or concurrent modification requires a new review of the differences
   and authorization for that revised proposal.

## Apply, reread and continue

7. Call `atlas_interact_apply_workunit_program_changes` with `workUnitId`, the
   chosen `plcId`, the baseline `expectedRevision`, that PLC's read pin as
   `expectedStandardVersionId` (including a read `null`), and 1–100 authorized
   `changes`. Keep dependent changes together in one bounded batch. If a
   dependent set cannot fit, revise the proposal before any write; do not split
   it into invalid intermediate documents. Several independent calls are not
   one transaction.
8. For a confirmed domain refusal, report its code, reason and locations as
   the [conduct reference](references/conduct.md#refusals-and-locations)
   describes. A revision or pin conflict requires fresh reads and a new review
   of the remaining differences; never silently merge or automatically resend.
   For a lost mutation response or dependency error, the outcome **may be
   unknown**: stop dependent writes and follow shared recovery, rereading this
   Work Unit and its PLC before any further write. If reads remain unavailable,
   leave the step blocked. Do not replay a create or replace its expected
   revision with a newer one merely to make it succeed.
9. After every successful mutation, reread with `includeEditDetails: true`
   before another write. Compare the observed state with the authorized
   proposal, including content not targeted. Use the returned `changed`,
   revision, affected elements and diagnostics, then the reread, to report
   saved state and remaining work; a difference may come from another writer.
   For independent steps that fail later, retain the effects confirmed by
   earlier rereads and distinguish saved, refused, unknown and unfinished.
   A valid `changed: false` keeps the revision; a stale revision still conflicts.
10. If the request reaches generability, use
    [configure-plc's composition check](../configure-plc/SKILL.md#check-whether-the-plc-is-generable)
    and `atlas_interact_check_plc_composition` on the verified final PLC. Follow
    shared restitution for every returned diagnostic and any unnamed remainder
    according to `totalDiagnostics`. Never derive generability from a save or
    reimplement the generator's rules. Saved, incomplete draft and generable
    are distinct results; none compiles, publishes, deploys or commands a PLC.
    After a confirmed reread, use [open-page](../open-page/SKILL.md) for the
    verified Work Unit link. The browser and consultative MCP App stay optional.
