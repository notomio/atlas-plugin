---
name: configure-conditions
description: Use when reading, proposing or replacing the initial conditions (CI, start permissives) and permanent conditions (CP, interlocks) of an Interact equipment in Atlas, from a functional analysis or a dictated request. Triggers on "set the start permissives of this pump", "add an equipment interlock", "CI/CP de cet équipement" or "saisir les conditions d'équipement depuis l'AF". Clarify an unqualified CI/CP request; phase or Work Unit conditions use configure-workunit-program. Reads first, writes only the full document the user confirmed; never generates or deploys.
---

# Configure equipment conditions

Name the equipment as owner before proposing any write. If "CI/CP" leaves the
owner unclear, ask equipment or phase and which entity. For a Work Unit's
phase CI/CP, follow
[configure-workunit-program](../configure-workunit-program/SKILL.md) with its
required tools; never send phase conditions to the equipment condition tools.
For an explanation alone, use [interact-guide](../interact-guide/SKILL.md)
without any mutation.

Read the [connector and recovery procedure](../configure-plc/references/session-and-recovery.md),
the [conditions contract reference](references/conditions.md) and
[restitution](../configure-plc/references/restitution.md). Render every read,
effect and refusal as restitution prescribes. Read repères with
[ISA-5.1 letters and equipment codes](../interact-guide/references/isa-5-1-tags.md)
— a suggestion to confirm against the site convention, never a fact. Use
[interact-guide](../interact-guide/SKILL.md) for explanations.

Keep the chosen connector throughout. Several calls are not a transaction: say
so when a workflow spans more than one write. A document the user supplies
stays in this conversation; only structured clauses travel through the tools.
Everything about the standard — objects, UDTs, fields — is read in the session,
in the version the equipment's PLC pins, never recalled or assumed.

## 0. Check the tools before anything else

Inspect the connected session for `atlas_interact_get_equipment_conditions`,
`atlas_interact_set_equipment_conditions`,
`atlas_interact_get_equipment_configuration` and `atlas_search`.

**If `atlas_interact_get_equipment_conditions` is absent, stop here** — before
reading the document for proposals and before proposing any condition. Name the
missing tool, the connector and the blocked step, and ask for the current Atlas
connection to be restored. Without the read, a write would silently delete what
the inspector holds. If `atlas_interact_get_equipment_configuration` or
`atlas_search` is absent, stop as well: without them no operand can
be resolved from the session. If only the write tool is absent, you may read
and render the current conditions, but propose no write.

## 1. Resolve the equipment and read its configuration

Find the equipment with `atlas_search` and `types: ["equipment"]`. In
`results`, verify `type`, `id`, `siteId`, `identifier` and `target`; `totals`
counts all matches per type. `siteId` only favors that site in the tenant-wide
ranking. Ask when several match, and use the exact `identifier` for any later
`physicalId` input. Then read
`atlas_interact_get_equipment_configuration` with its `equipmentId`, without a
cursor. Retain the PLC, the pinned version, `selectedObjectTypeKey`, the
selected object's exchange UDT, every entry of `objects` and
`availableEquipment`.

Follow `nextChannelCursor` as `channelCursor` until it is `null`, collecting
every channel page **before** declaring any channel absent. If `plcId` or
`standardVersionId` differs from the first page, discard the pages and restart
without a cursor. With no PLC, no pin or no selected object, name that
prerequisite and route to [configure-plc](../configure-plc/SKILL.md) or
[configure-equipment](../configure-equipment/SKILL.md); stop before proposing.

## 2. Read the current conditions

Call `atlas_interact_get_equipment_conditions` with the `equipmentId`. This read
is the **baseline**: every condition it returns is kept, corrected or discarded
by an explicit decision, never lost by omission.

- `supported: false` means conditions cannot be written yet: no PLC, a pinned
  version that does not resolve, no selected object, or an object that does
  not declare the `STATE_INIT` / `STATE_PERM` pair in that version. Tell which
  from the configuration read of step 1 — including its
  `unavailableReferences` — name that prerequisite, route to
  `configure-equipment` or `configure-plc`, and stop before proposing. The
  only write left is clearing both lists, which the server accepts without
  that prerequisite: offer it only if leftover conditions are listed, and only
  after confirmation of what it deletes.
- Render the current conditions list by list, as restitution prescribes.
- A row whose `operandRef`, `comparisonOperandRef` or `rangeOperandRef` is
  `null` while its id is present is a **reference unavailable**. Keep that
  condition unchanged in the document unless the user explicitly decides
  otherwise, and warn now that resending it as it stands is refused with
  `NOT_FOUND` and writes nothing — the new conditions included. It blocks the
  write of step 6 until the user drops or repoints it.
- A rank listed in `positions` with no row is a condition with no clause. Say
  so; it cannot be resent as it stands, so the user decides its fate.

## 3. Propose each condition

Read the functional analysis or the request sentence by sentence. For each
safety or permissive it states, propose one condition: its list, its label, its
junction and its clauses, following the [clause forms](references/conditions.md#clause-forms).

- **List.** A condition that must hold before the equipment may start belongs
  to `init`; one that must keep holding while it runs belongs to `perm`. When a
  sentence does not say which, or reads as both, ask.
- **Sense.** A condition states what must be TRUE for the equipment to start or
  keep running. A prohibition in the document ("shall not start when …") is
  written as its opposite; say that inversion in plant words and have it
  confirmed.
- **Operands.** Resolve each operand to a channel of this PLC's pool, or to a
  field of an equipment listed in `availableEquipment`, as the
  [operand reference](references/conditions.md#resolve-an-operand) describes —
  names, types and fields exactly as the reads of step 1 returned them.
- **Gaps.** When a repère, channel or field resolves in no read of this
  session, render the gap as restitution prescribes, with the words that did
  not resolve and what was searched. Write no clause in its place, and withhold
  the whole new condition from the document — a partial `all` would weaken the
  interlock. When the gap hits a correction of a baseline condition, keep the
  baseline version unchanged instead of dropping it, and say so. If the source is not in the
  pool, [link-plc-data-source](../link-plc-data-source/SKILL.md) is the separate
  action that adds it; then restart at step 1.
- **Units.** A threshold is a constant in the unit of the field it compares,
  read from the channel's unit and range. If the document states another unit
  or an unknown scale, ask; never convert silently.
- **Unreadable input.** When a sentence has no identifiable repère, or a
  threshold or sense is ambiguous, name what is missing and stop on that
  condition rather than completing it. A gap is a result.

## 4. Render the proposal

Render every proposed condition as restitution prescribes, the source sentence
included whenever a document was supplied. That rendering is what the user
confirms in step 5.

## 5. Confirm the whole document

The write replaces **both lists in full**. Present the complete resulting
document, as in the [review example](references/conditions.md#review-the-whole-document):

- **kept** — conditions from the baseline, unchanged, resent from the read;
- **corrected** — baseline conditions whose content changes, before and after;
- **new** — conditions proposed in step 3;
- **discarded** — baseline conditions the write will delete;
- **withheld** — gaps, which the write will not contain;

with the final list and position of every condition, and the warnings from
step 2. Wait for the user's **explicit confirmation of this whole document**.
Any change afterwards — one clause, one position — means presenting the
document again and waiting again.

Do not write while the document still holds a condition with a reference
unavailable: the refusal is certain and would take the whole document with it.
Ask the user to drop or repoint it; keeping it means no write, and say so.

Immediately before writing, read `atlas_interact_get_equipment_conditions`
again. If it differs from the baseline, someone else changed the conditions:
rebuild the document on this new read, present it and wait for confirmation
again. This narrows the window without closing it; say so.

## 6. Write, then handle a refusal

Only after that confirmation, call `atlas_interact_set_equipment_conditions`
once, with the confirmed document built as the
[write reference](references/conditions.md#build-the-write) describes.

- **`CONDITIONS_INVALID`** — nothing was written. Render **each** diagnostic on
  the condition and clause it names, by list and position in the document you
  submitted, in plant words, following the
  [diagnostic reference](references/conditions.md#diagnostics). Propose the
  correction, present the corrected whole document, obtain confirmation again,
  then resend **both lists in full**.
- **`CONFLICT`** — nothing was written. Read the returned reason. A missing
  prerequisite (PLC, pinned version, object, or the state pair) is named and
  routed to `configure-plc` or `configure-equipment`. A reason saying the PLC
  association or the pinned version changed meanwhile means the context moved:
  restart at step 1 and rebuild the document before asking again.
- **`NOT_FOUND`** — nothing was written. The equipment, or a channel or
  equipment a clause names, no longer exists; the refusal names only the kind
  of resource, not which one. Read the conditions and the configuration again
  before naming anything, then name the conditions whose reference is now
  unavailable — or the missing equipment itself — and let the user drop or
  repoint them.
- **Timeout or lost connection** — the outcome is unknown; do not say nothing
  was written. Restore the same connector and read the conditions before any
  new write, as the recovery procedure prescribes.

## 7. Reread and report

After a successful write, read `atlas_interact_get_equipment_conditions`. Compare
it with the confirmed document: counts, order and every clause. Report a
difference rather than correcting it silently; a concurrent writer may explain
it.

Report, as restitution prescribes, what the reread shows is written, what was
discarded, and what was withheld as a gap. For each written condition, give its
inspector target — the row's `target` value, `<equipmentId>:<init|perm>:<position>`,
copied from the reread and never composed. If the user asks to open a condition
or receive its full link, follow [open-page](../open-page/SKILL.md) with the
verified equipment, site and returned target. The app origin comes from the
plugin [README](../../README.md), never from the connector's MCP or API URL;
when the README documents no app origin, give the target alone.

Saved conditions say nothing about whether the PLC is generable, deployed or
what the machine does. This workflow does not generate, compile, deploy or
publish.

To answer whether the PLC is generable, follow
[configure-plc](../configure-plc/SKILL.md#check-whether-the-plc-is-generable).
