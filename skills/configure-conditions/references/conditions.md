# Read and replace equipment conditions

The connected tool schemas are the payload authority. These field names describe
the integrated Interact tools; if one is absent, stop its dependent step using
the [recovery procedure](../../configure-plc/references/session-and-recovery.md).
How a condition is worded to the user belongs to
[restitution](../../configure-plc/references/restitution.md) alone; this file
maps reads to writes and does not restate its rules. It supplies no equipment
ids, standard, UDT or field names: every one comes from a read of the session.

## What a condition means

Each list is one aggregate: all its conditions joined by AND, and an empty list
is TRUE — the equipment starts, or keeps running, without restriction. A
condition is a group of clauses joined by its `junction`: `all` (AND) or `any`
(OR). A condition is TRUE when the equipment is allowed; a prohibition in a
document is therefore written as its opposite.

## Read the current lists

`atlas_interact_get_equipment_conditions` takes `equipmentId` and returns
`equipmentPhysicalId`, `supported`, `positions` (the ranks of each list) and
`rows` — **one row per clause**, never one per condition. A row carries `kind`
(`init` or `perm`), `position` (the condition's 1-based rank in its list),
`clausePosition`, `label`, `junction`, `negated` and `target`.

Each operand slot uses five columns: `operandKind` (`io_channel` or
`equipment`), `operandId`, `operandRef` (the channel tag or equipment repère
the server resolved, `null` when that reference no longer exists),
`operandDataType` (the equipment's exchange UDT, `null` for a channel) and
`operandField`. The comparison value repeats them as `comparisonOperand…`, the
range value as `rangeOperand…`. A comparison also carries `comparisonOperator`,
`comparisonValueKind` (`constant` or `field`) and, for a constant,
`comparisonConstantType` with exactly one of `comparisonBooleanValue`,
`comparisonNumberValue` or `comparisonStringValue`. Range columns mirror them
under `range…`. Any field a clause omits is `null`; `negated` is always a
boolean, `false` when the clause does not negate.

Group rows by `kind` and `position`, and order the clauses of a group by
`clausePosition`. A rank in `positions` with no row is a condition with no
clause: its label and junction are not in the read.

## Resolve an operand

Every name below is copied from a read of step 1, in the version the PLC pins.

| Operand             | Where it comes from                                                                                                                                         | Written as                                            |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| Channel of the pool | A channel from the configuration pages, found by its `tag`; the field is one of its `exchangeDataTypeFields`                                                | `{ kind: "io_channel", channelId, field }`            |
| Field of a peer     | An entry of `availableEquipment`, found by its `physicalId`; its `objectTypeKey` names an entry of `objects`, whose `exchangeDataTypeName` and fields apply | `{ kind: "equipment", equipmentId, dataType, field }` |

`availableEquipment` holds the other equipment of this PLC whose object exists
in the pinned version; one absent from it is not an operand. A channel's
`active` or `pointActive` at `false` is said to the user, not hidden. For an
analog value, establish whether the raw or the engineering field is meant, as
the [signal reference](../../interact-guide/references/signals.md) explains,
and choose among the fields actually returned. A document's words that match
none of these reads are a gap, whatever their resemblance to a returned name.

## Clause forms

The resolved field's type decides the clause shape.

| Field type                                                               | Clause                                                                                                                                                           |
| ------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `BOOL`                                                                   | `operand`, optional `negated: true`; no `comparison`, no `range`                                                                                                 |
| `SINT` `INT` `DINT` `LINT` `USINT` `UINT` `UDINT` `ULINT` `REAL` `LREAL` | `operand` and `comparison` (`operator`: `=` `<>` `>` `<` `>=` `<=`); no `negated`; an optional `range` only after `>`/`>=`/`<`/`<=`, with the opposite direction |
| Anything else                                                            | Not an operand: say so, write no clause                                                                                                                          |

A comparison or range `value` is either a constant or another field:

- constant: `{ kind: "constant", literal: { type, value } }`, where `type` is
  the operand field's exact IEC type — `value` is a decimal string for `LINT`
  and `ULINT`, a number otherwise. An on/off signal never takes a constant: it
  is read plain or `negated`;
- field: `{ kind: "field", operand: <an operand as above> }`, of exactly the
  same type.

A range bounds a value on both sides: `>= 0.5` with range `<= 3.8` reads
"between 0.5 and 3.8".

## Build the write

`atlas_interact_set_equipment_conditions` takes `equipmentId`, `version: 1`,
`init` and `perm` — at most 100 conditions per list, in the confirmed order.
Each condition is `{ label?, junction, clauses }`.

A **kept** condition is rebuilt from its rows, never retyped:

- `label` from the row; `null` means **omit the key** — the write refuses an
  explicit `null`. A label holds 1 to 120 characters; a longer one is refused
  as invalid input, before any condition is checked;
- `junction` from the row; `negated: true` only when the row says `true`;
- each operand rebuilt from its columns: `operandKind` becomes `kind`,
  `operandId` becomes `channelId` (for `io_channel`) or `equipmentId` (for
  `equipment`), `operandDataType` becomes `dataType` (for `equipment` only)
  and `operandField` becomes `field`; `operandRef` is display only and never
  sent. The `comparisonOperand…` and `rangeOperand…` columns map the same way;
- a constant `literal` as `{ type: <…ConstantType>, value: <the one non-null
  value column> }`; a field value from its `…Operand…` columns;
- a `null` comparison or range column group means that key is absent.

A condition with a reference unavailable is never sent: the write would answer
`NOT_FOUND` and save nothing. It stays in the reviewed document until the user
drops or repoints it, and the write waits for that decision.

## Diagnostics

`CONDITIONS_INVALID` lists one diagnostic per failing clause, and one per
condition that fails as a whole: `kind`, `position`, optional
`clausePosition` and `code`. `clausePosition` is present when a clause fails
and absent when the condition itself does (`EMPTY_CONDITION`). Positions count
in the document **you submitted**. The whole write was refused. One condition
can appear several times — render each diagnostic.

| Code                           | What to tell the user, and the correction                                                  |
| ------------------------------ | ------------------------------------------------------------------------------------------ |
| `EMPTY_CONDITION`              | The condition has no clause: add one or discard it                                         |
| `OPERAND_NOT_IN_PLC_POOL`      | The channel's data source is not attached to this PLC: link it, or choose a pooled channel |
| `OPERAND_EQUIPMENT_NOT_ON_PLC` | The named equipment is not on this PLC: choose one from `availableEquipment`               |
| `OPERAND_UNKNOWN_FIELD`        | The field, or the equipment's UDT, is not in the pinned version: choose a returned field   |
| `OPERAND_TYPE_UNSUPPORTED`     | The field is neither boolean nor numeric: choose another field                             |
| `NEGATION_NOT_ALLOWED`         | A numeric value cannot be negated: state the opposite comparison                           |
| `COMPARISON_REQUIRED`          | A numeric value needs a comparison and a threshold                                         |
| `COMPARISON_NOT_ALLOWED`       | An on/off signal takes no comparison: keep it plain or negated                             |
| `RANGE_NOT_ALLOWED`            | A range needs a numeric value and a `>`/`>=`/`<`/`<=` comparison                           |
| `RANGE_OPERATOR_INVALID`       | The two bounds point the same way: the range must close the comparison from the other side |
| `CONSTANT_TYPE_MISMATCH`       | The threshold's type differs from the field's: restate it in the field's IEC type          |
| `FIELD_TYPE_MISMATCH`          | The compared field has another type: choose a field of the same type                       |

Name the condition by its repère, list, position and label, and the clause by
its position, as restitution prescribes; the code may follow indented.

## Review the whole document

A hypothetical review before confirmation. Every entry opens on the
equipment's repère; IEC names and ids only follow indented:

> **PU101 — start permissive, list `init`, position 1, kept unchanged**
> TOR: the tank low-level switch LSL101 must read clear; its channel is wired
> direct.
>
> **PU101 — start permissive, list `init`, position 2, new, 2 clauses, all required**
> Clause 1 — TOR: the dry-run switch LSL102 must read clear; its channel is
> wired inverted, so the field reads the opposite of the raw contact.
> Clause 2 — ANA: level LIT101 at or above 0.5 m (range 0–4 m).
> Source (functional analysis, § 3.2): "The pump shall not start when the dry
> run switch is active or the measured level is below 0.5 m."
>
> **PU101 — running interlock, list `perm`, position 1, corrected**
> ANA: level LIT101 at or below 3.8 m (was: at or below 3.9 m).
> Source (functional analysis, § 3.4): "The pump shall stop above 3.8 m."
>
> **PU101 — running interlock, list `perm`, position 2, kept unchanged — reference unavailable**
> Clause 1 reads a channel that no longer exists. Resending this condition as
> it stands is refused (`NOT_FOUND`), and the new conditions with it: drop it
> or repoint it before anything is written — your decision.
>
> - channel id `<the operandId from the read>`
>
> **PU101 — discarded: former list `perm`, position 3, "manual override"**
> The write deletes it.
>
> **PU101 — withheld, gap**
> Source (functional analysis, § 3.5): "The pump shall stop when valve
> 101-XV-120 is closed." No channel or equipment of this PLC resolves
> `101-XV-120`. No clause is written.
>
> - IEC, for Beremiz: `IO_LSL101`, `IO_LSL102`, `IO_LIT101`.

## Inspector target

Each reread row carries `target`, `<equipmentId>:<init|perm>:<position>`: the
equipment, the list and the condition's rank, shared by every clause of that
condition. Copy it from the reread after the write — positions shift when a
condition is added, removed or moved. It names a condition in the Atlas web
app; opening the inspector on it also needs the site and the equipment
selection, so it is not a ready-made link.
