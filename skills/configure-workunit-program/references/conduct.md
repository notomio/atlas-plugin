# Conduct-specific reference

The connected tools' schemas are the payload authority. Read current ids,
interfaces and pins; the examples here are hypothetical, not calls or tenant
fixtures. Shared session/recovery and rendering remain in the references linked
by the [skill](../SKILL.md).

## Equipment versus phase

Equipment CI/CP use the existing equipment condition tools through
[configure-conditions](../../configure-conditions/SKILL.md). Phase CI/CP are
part of the Work Unit's conduct and use targeted phase-condition operations.
Name that owner first. Conditions read equipment/channel UDT fields on the
same PLC; conduct actions command only eligible equipment fields of this
Work Unit on that PLC. `commandable` is read from the pinned standard;
`actionTargetable` also establishes the action's scope and supported type.

Prêt (`0`) and Verrouillé (`-1`) are system states, not editable running states.
System transitions are not editable. A link carries one sequence or is immediate;
a sequence owner is `{ kind: "link", linkId }` or `{ kind: "abort" }`. The abort
is reserved `-2`, has priority, resets other Grafcets and ends in Verrouillé.
A CP lost while running triggers abort; CI are start permissives, not continuous
monitoring. These meanings do not authorize runtime commands from this skill.

OU branches are ordered transitions **within one step**: list position is their
priority. Keep their order unless the request changes it. Conditions in each
phase list combine with `AND`; clauses inside one condition/receptivity use
`junction: "all" | "any"`. A held condition uses `heldForMs`; an elapsed-step
wait uses `{ kind: "step_elapsed", durationMs }`. Durations are milliseconds:
distinguish zero, missing and explicitly incomplete `null`, and clarify units
before converting a user's duration. No ET divergence or nested sub-Grafcet.

## Use the typed read

Join `actionValues` to `actions` by `actionId`: keep `constantType` and the
matching `booleanValue`, `numberValue` or exact decimal `stringValue` for
`LINT`/`ULINT`. A field-valued action keeps the operand fields from its action
row, rather than a constant inferred from display text. A missing target or
value stays incomplete unless its correction was requested.

Join `receptivities` by `transitionId`, retaining owner, step, label,
`hasReceptivity` and junction, then ordered `receptivityClauses` by
`clausePosition`. Keep operand/comparison/range columns, negation, `hasHold`,
`heldForMs` and `durationMs`. `hasHold: false` means omit `heldForMs`;
`hasHold: true` with a null duration is a hold still to fill. A present
receptivity with no clauses is an empty group, not an absent receptivity.

`phaseConditionHeaders` preserves each list (`kind: init | perm`), position,
label, junction and `clauseCount`, including empty conditions. Join its typed
clauses from `phaseConditions` by list/position and retain clause order. Never
drop a header because it has no clause row. A missing resolved name with an id
is an unavailable reference, not permission to replace it with a similar one.
Retain its diagnostic. `editDetailsIncluded: false` is not an empty program.

## Targeted changes

The envelope is `workUnitId`, `plcId`, `expectedRevision`,
`expectedStandardVersionId`, `changes`. The positive safe revision or null and
the pin come from reads; null revision creates only if no conduct exists.
The server initializes version 1 with Prêt and Verrouillé. Reassignment to a
different same-site PLC revalidates the entire retained document and must be
part of the reviewed proposal.

Changes use the `operation` discriminator, not JSON paths or endpoint names:

| Target                 | Operations and selection                                                                                                                                |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Running state / link   | `createState`, `updateState`, `deleteState` by `stateId`; `createLink`, `updateLink`, `deleteLink` by `linkId`                                          |
| Link or abort sequence | `createSequence`, `updateSequence`, `deleteSequence` by `owner`                                                                                         |
| Step                   | `createStep` with `owner` and new `stepId`; `updateStep`, `deleteStep` by `stepId`                                                                      |
| Action                 | `createAction` with `stepId` and new `actionId`; `updateAction`, `deleteAction`, `moveAction` by `actionId`                                             |
| Transition             | `createTransition` with `stepId` and new `transitionId`; `updateTransition`, `deleteTransition`, `moveTransition` by `transitionId`                     |
| Phase CI/CP            | `insertPhaseCondition`, `replacePhaseCondition`, `deletePhaseCondition`, `movePhaseCondition` by `conditionKind` and position as required by the schema |

Existing identities come from the conduct read. Assign a fresh UUID to each
new element once, keep it stable in the reviewed batch and its references,
and never substitute example identities. Creation with an existing identity is
refused, not an update. Omit numbers to use server allocation unless a number
was requested; do not renumber other elements to make room.

Omitted update properties are preserved. Use null only on fields the schema
allows to be cleared. Replacing a condition or a transition's receptivity
replaces **that entire group**, including label, junction and all clauses:
carry unchanged parts from typed reads. A one-clause correction is not a
one-clause replacement of a larger safety group. Positions start at 1 and
refer to the intermediate list as the batch advances; `toPosition` is the
final position within that same list. Do not resend the full conduct.

Review deletion effects explicitly: deleting a link removes its sequence;
deleting a sequence removes its steps/actions/transitions and makes its link
immediate; deleting a step removes its actions and outgoing transitions and
can clear the initial step. Other incoming references are not silently
redirected or deleted: correct/remove their uses explicitly in the same batch
or accept refusal. Removing abort can leave a non-publishable draft. Deleted
elements lose their layout; a reconnected link/transition loses its old bends.
The remaining layout is preserved, without global rearrangement.

## Three hypothetical examples

These are change shapes, with symbolic placeholders to replace from current
reads or new identities. They are not executable JSON and require no local
script. The surrounding envelope and every field follow the connected schema.

**Add a step.** For a link sequence already read, assign a fresh step identity:

```text
{ operation: "createStep", owner: { kind: "link", linkId: <read link id> },
  stepId: <new step id>, label: "Wait for availability" }
```

This appends a step without an implicit action, transition or initial-step
change. If the approved proposal includes making it initial, add
`updateSequence` with that owner and `initialStepId: <new step id>` in the same
batch. A requested wait can use `createTransition` with a new transition id,
that `stepId`, an explicitly chosen target (`kind: "step"` with `stepId`, or
`kind: "end"`) and `receptivity: { junction: "all", clauses:
[{ kind: "step_elapsed", durationMs: <confirmed milliseconds> }] }`.
Check how the new step is reached from the initial step. Inserting it before
an existing end also requires an authorized `updateTransition` pointing the
chosen incoming transition to it, preserving that transition's receptivity
and OU position. Creation alone does not reconnect the existing sequence.
Review any missing continuation as a draft gap; do not invent one.

**Correct one phase CP.** At a read `perm` position, one numeric threshold
changes. Use `replacePhaseCondition` with `conditionKind: "perm"`, that
`position`, and `condition` reconstructed from the read header and **all** its
typed clauses. Change only the selected comparison's constant literal
(`kind: "constant", literal: { type: <read IEC type>, value: <confirmed value> }`);
keep label, junction, operands, second bound and other clauses. The other
conditions and the equipment CI/CP are untouched. Present old/new values and
units before applying; an unresolved safety clause is never omitted.

**Concurrent edit.** A batch proposed on a read revision is refused with
`revision_stale` after an Atlas editor save. Reread the conduct, PLC/pin
and needed interfaces, compare with the original baseline, then present only
the remaining differences for renewed review. Do not substitute the new
revision and resend the old batch. If the requested state is already present,
report the reread instead of creating a second element. The same
reconciliation is needed after a lost response: a repeat with the stale
revision is refused even if its content would now be unchanged; changing ids
or using null again is not a recovery strategy.

## Refusals and locations

A confirmed targeted domain refusal leaves that batch's document, PLC and
revision unchanged, including its deletions. Prior independent calls stay
saved. A dependency/transport error does not establish such a refusal; use
the skill's unknown-result recovery before any further write.

- `revision_stale` or `standard_version_changed`: refresh context and
  review differences again, never an automatic merge/retry.
- Other `CONFLICT` reasons (missing target, duplicate identity/number, invalid
  position, deletion still referenced or dangling reference): locate the
  `changeIndex` (zero-based), returned `usages` or `locations`. Explain the
  affected sequence owner, element or phase list/position, then revise only
  with a new review. Do not remove a safety clause to bypass the refusal.
- `NOT_FOUND`: use `resource` and any returned selectors to reread the missing
  Work Unit, PLC or reference. A scope refusal is not permission to move it.
- Validation failure: name the rejected field/index and keep the batch
  unapplied; use the actual input schema's bounds.
- Successful save with diagnostics: report each returned code, reason and
  location — owner/link, state, step, action/transition, or phase condition
  list/position/clause. Missing publishability is not a failed persistence.
  Neither a clean save nor absence of conduct diagnostics proves composition;
  use the existing PLC check through configure-plc when requested.
