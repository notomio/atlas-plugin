---
name: configure-plc
description: Use when preparing an existing Interact PLC in Atlas, choosing or adopting a published standard version, removing its pin, or preparing its source pool before equipment or Work Unit conduct configuration. Triggers on "prepare this PLC", "pin the standard", "adopt version", "remove the pin", "configure le PLC", "choisir la version du standard", "préparer les sources du PLC", "is this PLC generable", "est-il générable", or "composition à blanc". Route phase/GRAFCET edits to configure-workunit-program. Does not create or deploy a PLC.
---

# Prepare an existing PLC

Read the [connector and recovery procedure](references/session-and-recovery.md)
before calling tools. Keep the chosen connector throughout, including when
reusing [link-plc-data-source](../link-plc-data-source/SKILL.md). Explain concepts
with [interact-guide](../interact-guide/SKILL.md) as needed. Render every read,
effect and refusal as [restitution](references/restitution.md) prescribes.

## Discover and preserve existing choices

1. Resolve the site with `atlas_isa95_list_sites`, then call
   `atlas_interact_plc_list` with its `siteId`. Select the requested PLC from
   `plcs`; ask if several match. An empty list means a PLC must first be created
   in Atlas. Never manufacture a PLC or its id.
2. Record its `id`, `name`, `standardVersionId`, `equipmentIds` and `dataSourceIds`.
   Keep an existing pin unless the user asks to change or remove it. An existing
   source pool is retained while adding requested sources.
3. If pinning is necessary, or a version change is requested, use
   `atlas_interact_list_standard_versions` with `{}`. Present `versions` with
   `id`, `version` and `createdAt`. Map the requested version to its returned id.
   With no explicit target, **wait for the user's choice**. The newest-first
   order never authorizes selecting the first or latest version automatically.
   An empty catalogue is a prerequisite failure: a published version is needed.
   Do not substitute a draft or administrator snapshot.

## Pin or adopt a version

4. Ensure `atlas_interact_pin_plc_standard_version` and
   `atlas_interact_get_equipment_configuration` are exposed. Before changing the
   pin, read the configuration of **every** `equipmentId` listed on the PLC.
   Retain the full standard and IO binding collections from those reads for
   comparison (bindings are complete on every page).
5. Announce the target version and the adoption effects **before the call**:
   accepted adoption removes standard bindings for inputs absent from the target
   version; it preserves standard bindings for retained inputs, the equipment
   and their IO wiring. Incompatible adoption is refused with diagnostics.
   Resolve authorization for those effects, then call with the real `plcId`
   and explicitly chosen `standardVersionId`.
6. On `STANDARD_VERSION_ADOPTION_BLOCKED`, report **all** returned diagnostics,
   including equipment, input and reason where present. The refused adoption
   preserves the pin and bindings. Do not delete any binding to force it through.
   On other refusals, preserve the supplied code/data and follow recovery.
   Repeating the same version id preserves configuration; do not infer a
   `changed` flag from a pin response, which contains only the two ids.

## Remove a pin only when requested

For an explicit removal, read the PLC and its equipment first. If an equipment
still has a `selectedObjectTypeKey`, explain that its standard association blocks
removal. Do not clear its type or bindings as a workaround. Otherwise announce
that the PLC will become **non-publishable**, then request
`standardVersionId: null` with the pin tool. The server's existing guards decide
the result, including races since the read; report any `CONFLICT` unchanged.
No generation or installation accompanies a successful removal.

## Prepare sources and reread

7. After any successful pin/removal, reread `atlas_interact_plc_list` and the
   configurations of the union of equipment ids seen before and after the call.
   Verify the actual pin and refreshed interfaces. Compare standard bindings
   and IO wiring with the baseline. Report removed, retained or otherwise
   changed bindings as **observed differences**; concurrent writers may also
   have contributed. Report a missing equipment/reference instead of assuming
   it was deleted by adoption. If the pin differs from the requested one, stop
   and resolve the changed context before proceeding.
8. For each source to attach, follow
   [link-plc-data-source](../link-plc-data-source/SKILL.md) on the same connector.
   Reuse an existing source; create one only if missing and authorized. Source
   creation and linking are separate calls: preserve the created id if linking
   fails. A source already on another PLC is a conflict, not permission to move
   it. Never replace the whole pool to attach one source.
9. Reread the PLC and affected equipment after preparation. Report the selected
   version (or absent pin), source ids, observed binding effects and unfinished
   prerequisites. Linking sources and saving configuration do not prove that
   data is flowing. Continue with
   [configure-equipment](../configure-equipment/SKILL.md) only within the request.
   If the request includes phase CI/CP, states or GRAFCET sequences, continue
   with [configure-workunit-program](../configure-workunit-program/SKILL.md)
   once its required tools are exposed. Equipment CI/CP instead use
   [configure-conditions](../configure-conditions/SKILL.md); name the owner
   before proposing a write, and ask when equipment versus phase is unclear.

## Check whether the PLC is generable

Run this only when the user asks whether the PLC is generable — never to infer
a configuration rule: the rules live in the generator, and a diagnostic
describes one PLC at one moment.

10. Use the PLC `id` and `name` already read with `atlas_interact_plc_list` on
    this connector in the session; otherwise run discovery steps 1–2 first. No
    pin or source change is needed to ask.
11. Ensure `atlas_interact_check_plc_composition` is exposed, then call it with
    that `id` as `plcId`.
12. Render the answer as
    [restitution](references/restitution.md#rendering-a-dry-run-composition)
    prescribes, with the PLC name from that read.
13. Handle a refusal — none of them is a generability answer:
    - `NOT_FOUND` naming the PLC: the id is stale. Rerun discovery step 1 and
      select the PLC again; never guess an id.
    - `CONFLICT`: an equipment of this PLC is still used by Connect. Say so in
      plant words: its Connect acquisitions must be removed before Interact
      can drive it. Nothing was written.
    - Timeout or lost connection: the check persisted nothing and may be
      replayed on the same connector.
    - Any other refusal: report its code and stop; do not retry in a loop.

A diagnostic may lead, within the request, to
[configure-equipment](../configure-equipment/SKILL.md) or
[configure-conditions](../configure-conditions/SKILL.md) for equipment CI/CP,
or [configure-workunit-program](../configure-workunit-program/SKILL.md) for
conduct and phase CI/CP when its tools are exposed. The check deploys nothing.
