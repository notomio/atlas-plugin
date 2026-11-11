# Read and patch the current equipment contract

The connected tool schemas are the payload authority. These field names describe
the integrated Interact tools; if one is absent, stop its dependent mutation
using the [recovery procedure](../../configure-plc/references/session-and-recovery.md).
This reference supplies no equipment ids, object interface or register map.

## Equipment read

`atlas_interact_get_equipment_configuration` takes `equipmentId` and optional
`channelCursor` / `channelLimit`. It returns the equipment/site identity,
`plcId`, `standardVersionId`, numeric `standardVersion`, `selectedObjectTypeKey`,
`objects`, `availableEquipment`, `channels` and `nextChannelCursor`.

Each object exposes `key`, `mainBlockName`, `exchangeDataTypeName`,
`exchangeDataTypeFields`, `configurableInputs`, `outputVariables` and
`wiringSections`. Variables and fields carry flat `typeKind` / `typeName`,
their real `name` and documentation.

Use `wiringSections` to discover `setIo` targets. All four sections are returned
in declared order, including empty sections:

- `BLOCK_INPUTS`: block inputs, excluding `I_ACK`; direction `input`.
- `FROM_SCADA`: exchange UDT fields received from supervision; direction `input`.
- `TO_SCADA`: exchange UDT fields sent to supervision; direction `output`.
- `BLOCK_OUTPUTS`: declared block outputs; direction `output`.

Each section has `id`, `direction` and `variables`. Each variable also has
`direction` and a boolean `udtField` indicating whether it is an exchange UDT
field. Use its `name` verbatim as `setIo.varName`, including any parameter prefix,
and its returned `direction`. This boolean `udtField` is target metadata;
`setIo.udtField` instead takes the **name of the selected channel's UDT field**.

`configurableInputs` remains the source of targets for `setConstant` and
`setEquipmentField`. `outputVariables` is broader than the inspector offerings:
it also contains accepted `NOT_SENT` UDT commands used by the reference project.
These remain available for `setIo` with direction `output`; use their `name`
verbatim. Do not infer additional targets from `exchangeDataTypeFields`.

Saved origins are complete on every page:

- `constantBindings`: `varName`, `type` and one populated `booleanValue`,
  `numberValue` or `stringValue`; the other values are null. Preserve false
  and zero: they are real values. This is a **read shape**, not a patch literal.
- `equipmentFieldBindings`: `varName`, `sourceEquipmentId`, `dataType`, `field`,
  `sourceAvailable`, nullable `sourceLabel`.
- `ioBindings`: `varName`, `direction`, `ioManagerChannelId`, `udtField`,
  `channelAvailable`, nullable `channelTag`.
- `missingInputs` names unbound inputs. `unavailableReferences` names a resource
  and id to resolve; do not invent a label for it or delete it automatically.

Channel pages contain `id`, `tag`, `dataSourceId`, `dataSourcePointId`, `kind`,
`family`, `active`, `pointActive`, `unit`, nullable `direct`, `rawMin`, `rawMax`,
`engineeringMin`, `engineeringMax`, and the resolved exchange UDT name/fields.
The equipment read does not expose an address, NodeId or generated instance.
Four null analog bounds mean raw mode. The field selections are explained in
the [signal reference](../../interact-guide/references/signals.md).

## Patch command shapes

`atlas_interact_configure_equipment_bindings` takes exactly the target
`equipmentId` and `changes`. Each command uses `operation`, not `kind`.

| operation           | Other command fields                                     | Meaning                                                  |
| ------------------- | -------------------------------------------------------- | -------------------------------------------------------- |
| `setConstant`       | `varName`, `literal: { type, value }`                    | Typed literal for an actual configurable input           |
| `setEquipmentField` | `varName`, `sourceEquipmentId`, `dataType`, `field`      | Compatible field of a real peer equipment's exchange UDT |
| `setIo`             | `varName`, `direction`, `ioManagerChannelId`, `udtField` | Read or command a qualified signal's field               |
| `remove`            | `varName`                                                | Remove this variable's binding, whichever its origin     |

Use the exact current interface and schema. A BOOL literal carries a boolean;
LINT/ULINT carry canonical decimal strings to preserve 64-bit precision. Other
supported scalar literals carry numbers within the tool schema's bounds;
TIME is a non-negative integer duration in milliseconds. Do not round integers,
send an IEC expression as a number, or convert one scalar type to another.
No constant is offered for a non-scalar input.

For `setEquipmentField`, `dataType` is the source object's returned
`exchangeDataTypeName`, not the scalar field type or a constructed name.
For `setIo`, take direction from the returned wiring variable, or use `output`
for an additional command from `outputVariables`. It is not the SCADA direction
label or digital polarity. Send no
`dataSourceId`, protocol address, NodeId, generated instance or channel
configuration in this binding.

Each set replaces only that variable's origin; removals alone work even without
a PLC or object. A mixed set/remove patch requires the normal set prerequisites
and is atomic. Duplicate variable names (case-insensitive), `I_ACK`, an empty
batch and more than 100 changes are refused. An unchanged valid repeat returns
`changed: false`; `changedVarNames` and `missingInputNames` describe the result.
Read the saved state after success.

Type compatibility and point/channel activity are **selection filters** here.
Do not claim that the patch endpoint adds an inactive-channel or exact IO-type
validation: generation remains authoritative for its own diagnostics.

## Adjust a point or channel when necessary

Keep this work separate from equipment bindings. For a point name/address or
channel tag correction, use the targeted update tools below, including for
corrections across several rows. Do not route these edits through points sync.

`atlas_interact_list_channels` returns at most 100 channels per call (default
`limit: 100`). For each source, start without `cursor`, then pass each
`nextCursor` as `cursor` until it is `null`. Collect all pages before planning
corrections or reconciling a partially applied batch. Results are ordered by
stable channel id, so renaming a point or tag does not move it between pages.
Pages are live reads, not a snapshot: restart from the first page if channels
are added or removed during discovery.

To correct an entry that already exists, call `atlas_interact_list_channels`
with the actual `dataSourceId`. Each result pairs `channel` with `point`: use
`point.id` as `pointId` in `atlas_interact_update_point.rows[]` (optional `name`
and `address`, omitted fields stay unchanged), or `channel.id` as `channelId` in
`atlas_interact_update_channel.rows[]` (required `tag`). Both tools take
`{ rows: [...] }`, with 1–100 entries and each id only once. Use a one-row batch
for a single correction; split larger requests into bounded batches. The list
contains only points with an IO Manager channel; an unqualified raw point is absent, not necessarily
missing from the source. Do not create a replacement merely because it is absent.

Each row patches one entity atomically, in submitted order. A domain refusal
leaves that row unchanged, keeps earlier successes and does not stop later rows.
Inspect every result in `rows`: `applied` confirms acceptance (including an
unchanged valid patch); `errorCode` and `errorMessage` describe a refused row.
Invalid payloads, duplicate ids and out-of-bounds batches are rejected before
any write. If both points and channels need correction, use their separate
batches; neither batch rolls back the other. These tools expose no activity,
kind, unit or scaling edit; that narrow surface does not mean the backend forbids recalibrating a wired channel.
Renaming preserves bindings by id. An address correction must preserve the IO
kind matrix and, on a wired point, its counting family. A channel tag must be
unique case-insensitively across the site. Report a confirmed domain refusal
with its code and diagnostics; for an uncertain transport result, follow the
[recovery procedure](../../configure-plc/references/session-and-recovery.md).

If the required read or update tool is unavailable, stop the correction and
report the missing prerequisite. Do not fall back to sync or a raw endpoint.

## Import a register map

A register-map import or synchronization belongs to
[import-register-map](../../import-register-map/SKILL.md): it runs the preview,
the per-line confirmation, the per-batch apply and the row-by-row report. Do not
run `atlas_interact_preview_points_sync` / `atlas_interact_apply_points_sync`
from this skill.

That sync upserts on the point `name`, so it cannot rename a point, and it is
not a route for a targeted correction: use the update tools above. Bind a
channel it creates only after rereading the equipment configuration.
