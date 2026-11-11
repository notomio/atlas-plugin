---
name: configure-equipment
description: Use when progressively configuring an Interact equipment in Atlas, associating it with an existing PLC, selecting its standard object, setting typed constants, wiring IO or peer equipment fields, or removing individual bindings. Triggers on "configure this equipment", "wire this motor or valve", "set this input constant", "replace this input's source", "configure cet équipement", or "câbler les entrées et commandes". Preserves unrelated inputs and outputs; never deploys.
---

# Configure equipment progressively

Read the [connector and recovery procedure](../configure-plc/references/session-and-recovery.md),
[binding contract reference](references/bindings.md) and
[restitution](../configure-plc/references/restitution.md). Use
[interact-guide](../interact-guide/SKILL.md) for explanations. Check that the
required tools and verification reads exist on the selected connector **before
their dependent mutations**. Preserve this connector in every reused skill.

## 1. Discover the equipment and PLC

Resolve the site with `atlas_isa95_list_sites` and the equipment with
`atlas_search` with `types: ["equipment"]` or a scoped
`atlas_isa95_get_workunit` read. In search `results`, verify `type`, `id`,
`siteId`, `identifier` and `target`; `totals` counts matches beyond this limited
list. Search is tenant-wide, even with `siteId`, which only favors that site.
Verify the owner through a scoped read and narrow an ambiguous/truncated search.
Use the exact returned `identifier` for a later `physicalId` input. A search
miss alone does not prove absence.

Create with `atlas_isa95_create_equipment` only when a scoped owner read confirms
it is missing and the user's request authorizes creation. Use the requested
identity and actual parent, then retain the returned id. After uncertain
transport, search/read that identity before retrying creation.

Read `atlas_interact_get_equipment_configuration` with `equipmentId`. Preserve
its saved bindings even if `plcId`, version or interfaces are absent. Identify
the requested existing PLC with `atlas_interact_plc_list`. If preparation is
needed, follow [configure-plc](../configure-plc/SKILL.md): keep an existing pin,
require an explicit choice when a pin is needed, stop on an empty catalogue.

## 2. Associate and choose the object

Before changing an association or type, read `equipmentIds` on the current PLC
and, for a move, the target PLC. Read the configuration of their peers that may
read this equipment and capture their bindings too. Announce these effects and
resolve the user's choice before calling:

- Moving to another PLC deletes this equipment's IO wiring and standard bindings
  owned by it **or referencing it as a source**, including bindings on peers.
  The type key is retained but may be unavailable in the target pinned version.
- Detaching (`plcId: null`) also clears the type key. Do not detach as a repair
  shortcut. Same-site, tenant and Connect-acquisition guards remain authoritative.
- Changing/clearing the object type deletes its owned and incoming-reference
  standard bindings, including those on peers; it **preserves IO wiring**.
- Repeating the same association or type preserves configuration.

Call `atlas_interact_associate_equipment` with the requested existing `plcId`
only if needed/authorized. Reread the equipment and affected peers after success.
Use `objects` from this newly read pinned context to present valid object keys;
keep the selected key unless a change is requested. Do not invent a key from an
equipment label or offer the internal IO blocks as business equipment types.
When a selection is needed, obtain the user's choice among the returned keys,
then call `atlas_interact_set_equipment_object_type` with `objectTypeKey`.
Reread again, including peers whose bindings may have changed. Report observed
differences without attributing all concurrent changes to this call.

An explicit detach/type removal can finish here. A patch containing only
requested removals may proceed without PLC/version/type; do not attach, pin or
select a type just to remove a saved binding.

## 3. Read interfaces and page through signals

Start with `atlas_interact_get_equipment_configuration` without `channelCursor`;
`channelLimit` is 1–100, default 100. Find the selected object in `objects`.
Use its ordered `wiringSections` to discover IO wiring targets, including
`FROM_SCADA` input targets. Keep empty sections and use each variable's returned
name and direction without inferring them from a prefix. `configurableInputs`
provides targets for constants and equipment fields. `outputVariables` also
provides accepted output targets beyond the inspector sections, including
`NOT_SENT` UDT commands. Read the real source equipment and channel exchange
fields. Exclude reserved `I_ACK` in every case.

For an IO choice, follow `nextChannelCursor` as `channelCursor` until null before
claiming a complete candidate list or that a signal is absent. On every page,
compare `plcId` and `standardVersionId` with the first page. **If either changes,
discard the accumulated pages and prepared choices and restart without a
cursor.** Also refresh when the selected type changes. Do not combine old and
new interfaces. A repeated cursor or missing necessary data blocks selection.
If context keeps changing, explain why the configuration cannot yet continue.

Every page contains all saved bindings; do not concatenate those collections.
Channel pagination is not a multi-call snapshot. Mutations revalidate the live
context; reread and rebuild affected choices after a conflict.

If a required source is absent from the PLC pool, use
[link-plc-data-source](../link-plc-data-source/SKILL.md) with the same connector,
then restart discovery. Do not mistake a later page or an inactive signal for
a missing source, and never create a replacement signal blindly.

## 4. Choose the requested origin and command fields

For each `configurableInputs` entry, ask for **constant or field value** when
unspecified. For a field, choose **IO signal or equipment**. A `FROM_SCADA`
variable instead uses `setIo` with its returned `input` direction; its presence
in `wiringSections` does not make it a `setConstant` or `setEquipmentField`
target. Preserve other variables. A constant has the exact input's scalar type;
a field must have the same `typeKind` and case-insensitive IEC `typeName`,
without numeric coercion.

For new IO bindings, require `active` and `pointActive` to be true. The source
must belong to this PLC. Read each channel's `exchangeDataTypeFields` from the
projection, never by constructing a UDT name. Inputs may read a compatible
DI/DO/AI/AO field; DO/AO reads describe the previous cycle's state. Outputs may
target only DO/AO, with the following field actually present and type-compatible:

| Command destination                  | Required field       |
| ------------------------------------ | -------------------- |
| DO                                   | `STATE_LOGIC`        |
| AO with four non-null scaling bounds | `VAL_SCALED_VALUE`   |
| AO with four null bounds (raw mode)  | `VAL_UNSCALED_VALUE` |

If the field is absent or incompatible, stop that binding and explain the
available interface. Do not use `STATE_RAW` for a DO command. For analog input,
establish whether raw or engineering value is wanted. Confirm the meaning of
an OPC-UA value before proposing scaling; avoid converting an already converted
value again. Follow the [signal reference](../interact-guide/references/signals.md).

Polarity, units and analog bounds remain channel settings. If a requested
channel adjustment is necessary, follow the point/channel procedure in the
[contract reference](references/bindings.md), then reread the channel. Never
implement inversion or scaling as equipment constants to bypass that step.

For equipment fields, use `availableEquipment` on this PLC and the returned
exchange UDT of its object (reread the source equipment if needed). A missing
source, type or field is a prerequisite to resolve, not a guessed name.

## 5. Apply a targeted patch

Show each variable's current origin and proposed change. Call
`atlas_interact_configure_equipment_bindings` with `equipmentId` and `changes`
using the actual schema: `setConstant`, `setEquipmentField`, `setIo`, `remove`,
discriminated by **`operation`**. See [exact fields](references/bindings.md).

Submit 1–100 commands, with no duplicate IEC variable names regardless of case.
For more changes, use separate authorized batches and reread between them. Sets
replace only the targeted variable's old origin across both collections. Other
inputs **and outputs** stay intact. Incomplete configuration is allowed:
`missingInputNames` reports what remains, not permission to invent defaults.

A confirmed rejected patch applies none of its commands, including any
removals. Report the complete diagnostic with the relevant variable/channel and
previously successful steps. Read again before preparing the corrected patch.
An uncertain transport result requires rereading before retrying; never fall
back to the removed whole-equipment replacement or roll back earlier batches.

## 6. Reread and report

After every successful patch, call `atlas_interact_get_equipment_configuration`
again. Verify the actual PLC/version/type and, for each targeted variable, its
typed constant, equipment source/field, IO channel/field/direction or absence.
Compare the three binding collections, including unrelated inputs and outputs,
with the baseline. Report a difference rather than silently correcting it.

Report `changed`, `changedVarNames`, the state observed on reread, remaining
`missingInputs` and unavailable references. On intermediate failure, name each
confirmed step, the failed/unknown operation and the next read or action needed
to resume. Configuration coverage does not establish generation readiness or
physical communication. This workflow does not generate, compile, deploy,
publish or implicitly undo configuration.

To answer whether the PLC is generable, follow
[configure-plc](../configure-plc/SKILL.md#check-whether-the-plc-is-generable).
