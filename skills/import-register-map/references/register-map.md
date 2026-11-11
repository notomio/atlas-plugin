# From a register-table line to a sync row

The connected tool schema of `atlas_interact_preview_points_sync` is the payload
authority. This reference says what each row needs, what to ask when the
document does not say it, and how to read the result. It supplies no register
map, no device default and no kind table. How to render a proposal or a result
is decided by [restitution](../../configure-plc/references/restitution.md), not
here.

## What a row carries

A row is one table line: `name`, `address`, optional `active`, and an optional
`ioManager` block. `name` is the point's identity: the sync upserts on it, so
keep the document's label and never rewrite it to make it fit. `ioManager`
holds `tag`, optional `unit` and `active`, and a `qualification` of `kind`
plus `configuration`. It appears only on a confirmed line.

The tool validates the whole call before any row is diagnosed: one malformed
value rejects every row of the call, with no per-row code and nothing written.
Check each value before composing.

- `name`: 1 to 120 characters once trimmed, no control character — a line
  break copied from a table cell is one. A label that does not fit is a
  question: propose a shortened name, show the original, and wait. Never
  truncate or join lines silently.
- `ioManager.unit`: 1 to 120 characters. An empty unit cell is an omitted
  `unit`, never `""`.
- `wordOrder` and `dataType`: only the pairs listed below.

Omitted `active` means active, on the point and on the channel. Omitted
`ioManager.unit` means no unit. Both apply on update too: a row that omits
them reactivates a deactivated point or channel and erases a stored unit. Say
so before a batch is confirmed.

## What the table must fix, and what to ask

Ask for each item the document leaves open, **before composing any row**. Ask
once for the whole table when the table states it nowhere, and let a line that
states its own value win. Never fill one from a habit, another device or a
previous import.

### Modbus TCP

| Row field      | What the document must say                                                                                                                                                   | Ask when                                                                                        |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `registerType` | `coil`, `discrete`, `holding` or `input` — often given by a column, a section or a function code (FC01 to FC04)                                                              | the line gives only a number                                                                    |
| `addr`         | the **0-based** offset inside that register table, 0 to 65535                                                                                                                | always, unless the document states its base: see below                                          |
| `dataType`     | `Boolean` for `coil` / `discrete`; for `holding` / `input` one of `Int16`, `UInt16`, `Int32`, `UInt32`, `Float`, `Double`                                                    | a word register has no data type, or says only "word" or "register"                             |
| `wordOrder`    | required on **every** Modbus address: `abcd` or `badc` on a one-register point, any of `abcd`, `badc`, `cdab`, `dcba` on a two- or four-register point (`Int32` to `Double`) | the document gives no byte and word order; ask once for the device, and never default to `abcd` |

**The addressing base is a question, not a conversion.** Atlas stores `addr`
0-based. Tables often use another notation: `40001` for the first holding
register, `30001` for the first input register, `1`-based offsets, or a
hexadecimal offset. The same table can mean `addr` 0 or `addr` 1 depending on
the vendor. Ask which base the table uses, restate one line converted with it
("`40012` becomes `addr` 11"), and convert only after that confirmation.

**A bit inside a word register cannot be declared.** A table that documents
status bits as `40010.3` or "register 10, bit 3" describes something this
model does not represent: `Boolean` exists only on `coil` and `discrete`.
Never map such a bit to a coil, a discrete input or a computed offset — the
device does not serve that address. Name those lines as a gap, and stop on
them.

A 32- or 64-bit value occupies two or four consecutive registers. Two rows
whose ranges overlap are refused (`ADDRESS_OVERLAP`); a table that lists the
second word of a value as its own line is a line to ask about, not to import.

### OPC-UA

| Row field   | What the document must say                                                                               | Ask when                              |
| ----------- | -------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| `nodeId`    | the full node id, verbatim (`ns=2;s=Line1.Tank.Level`)                                                   | a line gives only a browse name       |
| `dataType`  | `Boolean`, `SByte`, `Byte`, `Int16`, `UInt16`, `Int32`, `UInt32`, `Int64`, `UInt64`, `Float` or `Double` | the type is absent or not one of them |
| `direction` | `input` (the PLC reads it) or `output` (the PLC writes it)                                               | the node's access is not stated       |

Never translate a node into a Modbus register, and never assume a node already
supplies an engineering value: see
[data source, point and channel](../../interact-guide/references/points-and-protocols.md).

### The qualification of a confirmed line

| Kind        | `configuration`                                                                            | Ask when                                                                                                             |
| ----------- | ------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------- |
| `DI` / `DO` | `{ "direct": true }` for a direct contact, `{ "direct": false }` inverted                  | the document does not say whether the contact or the command is inverted                                             |
| `AI` / `AO` | four bounds `rawMin`, `rawMax`, `engineeringMin`, `engineeringMax`, or `null` for raw mode | the document gives neither the raw range and engineering range, nor states the value is already in engineering units |

`null` is raw mode; an identity pair is not. Their meaning is in
[signal meaning and selection](../../interact-guide/references/signals.md).
Never complete a partial set of bounds.

## Forming one kind proposal

Propose **at most one** kind per line, and say why in plant words.

1. **Capability narrows what is possible.** A boolean readable point can serve
   `DI`, a boolean writable one `DO`, a numeric readable one `AI`, a numeric
   writable one `AO`. Which register or node is readable or writable is in
   [data source, point and channel](../../interact-guide/references/points-and-protocols.md).
   A point both readable and writable — a coil, a holding register — can serve
   either kind of its value type, so capability alone never picks.
2. **The document picks what is plausible.** A section title ("Inputs",
   "Commands"), a label ("level", "run command", "fault"), a unit, a range or a
   read/write column is evidence. A repère's letters read through
   [ISA-5.1 letters and equipment codes](../../interact-guide/references/isa-5-1-tags.md)
   are a weaker hint; when they disagree with the document, follow the
   document and say so.
3. **Ask when the evidence is thin or conflicting.** A holding register that
   carries a measurement — a remote card exposing its analog inputs on holding
   registers — is a plausible `AI` and a plausible `AO`. The server accepts
   both; only the user knows which one is true. Ask, and propose none rather
   than guess.

A line with no confirmed kind is sent without `ioManager`. On a new point, it
stays a **raw point**. On a point that already has a channel, the sync leaves
that channel as it is: it never removes one, and the line is not a raw point.
A spare or reserved line is sent the same way, or left out if the user says so.

## Tags become IEC identifiers

A channel tag becomes `IO_<tag>` in the generated PLC project. `IO_<tag>`
must be an IEC 61131-3 identifier: after the prefix, only letters, digits and
underscores, 1 to 120 characters for the tag. A repère such as `101-LSL-111`
fails on its hyphens.

When a tag fails, propose a correction that changes as little as possible —
`101_LSL_111`, not a new name — show it beside the original and wait for the
user's answer. Keep the document's label as the point `name`: only the tag is
corrected.

Tags are unique across the whole site, case-insensitively: `LSL111` and
`lsl111` are the same tag, and so is a tag already carried by another data
source of the site. Two lines of one batch claiming the same tag are refused
in the preview (`DUPLICATE_TAG_IN_PAYLOAD`). A collision with a channel
already stored — on this source or another one of the site — is not predicted
by the preview and surfaces only on apply (`DUPLICATE_IO_TAG`).

## Batches

At most 100 rows per call. Split a longer table in document order, and keep
each batch's rows exactly as previewed when you apply it. Each batch is its
own call: nothing is transactional across batches.

**Check names across the whole table before splitting.** A name repeated
inside one batch is refused (`DUPLICATE_NAME_IN_PAYLOAD`), but a name repeated
across two batches is the same point: the second batch silently moves it, and
the first line is lost. Labels such as `Spare` or `Reserved` repeat often.
Name every repeated label and ask for distinct names before composing any
batch.

**Preview a batch only once the previous one is applied.** A preview compares
against what is stored; a batch previewed before its predecessor is written
cannot see an overlap or a name that predecessor creates.

## Reading a result row

The preview and the apply return the same row shape; an action means "would
happen" in the preview and "happened" in the apply. Report only from the
apply.

| Field                      | Read it as                                                                                  |
| -------------------------- | ------------------------------------------------------------------------------------------- |
| `pointAction`              | `create`, `update`, `unchanged` or `failed` for the point                                   |
| `pointErrorCode`           | why the point was refused, or `null`                                                        |
| `channelAction`            | `create`, `update`, `unchanged`, `failed` for the channel                                   |
| `channelAction: "none"`    | the row carried no `ioManager`; read it with `pointAction`, below                           |
| `channelAction: "skipped"` | the point half failed, so the channel was never attempted; the cause is on `pointErrorCode` |
| `channelErrorCode`         | why the channel was refused, or `null`; `null` on a `skipped` row                           |

`channelAction: "none"` says only that no channel was sent. Report the line
as a **raw point** when its point half succeeded and the source read showed no
channel on it. When that read showed a channel, say the existing channel was
left unchanged. When `pointAction` is `failed`, the line is refused and
nothing was written for it.

A point written with a refused channel is a real, partial result: the point
exists, with the channel it had before or none, and the line needs its channel
corrected. `pointErrorMessage` and
`channelErrorMessage` are server text: use them to understand, and name the
refusal by its code.

## Worked example

A remote I/O card table, Modbus TCP, on a source that holds no point yet. The table uses `1xxxx` / `3xxxx`
notation and gives no word order. Three lines:

| Address | Type         | Data   | Label                          | Unit | Range            |
| ------- | ------------ | ------ | ------------------------------ | ---- | ---------------- |
| 10005   | Discrete in. | Bool   | 101-LSL-111 tank 111 low level |      |                  |
| 30021   | Input reg.   | UInt16 | PU131 motor current            | A    | 0–27648 → 0–50 A |
| 30022   | Input reg.   | UInt16 | Spare                          |      |                  |

**Asked before composing:** the addressing base (the user confirms `10001` and
`30001` are offset 0); the device's byte and word order (the user confirms
`abcd`); the polarity of the low-level switch (the user confirms it is wired
direct).

**Proposed** — rendered as
[restitution](../../configure-plc/references/restitution.md) prescribes, one
kind each, the tag correction shown and awaiting an answer:

> **101-LSL-111 — tank 111 low-level switch, proposed as DI**
> TOR, discrete input 10005, wired direct. Addressing as you confirmed: `addr`
> 4, 0-based, `abcd`.
> The repère does not form an IEC identifier: channel tag proposed as
> `101_LSL_111`, point name kept as written. Confirm both, or give another tag.
>
> **PU131 motor current — proposed as AI**
> ANA, unit A, raw 0–27648 mapped to 0–50 A. Input register 30021: `addr` 20,
> `abcd`. Confirm and it also becomes channel `PU131_CURRENT`.
>
> **Spare — no kind proposed**
> Input register 30022, `addr` 21. It stays a raw point unless you qualify it.

The user confirms the first line and declines the second. The batch then
sends a qualified row and two raw rows. All three:

```json
[
	{
		"name": "101-LSL-111 tank 111 low level",
		"address": {
			"protocol": "modbus_tcp",
			"registerType": "discrete",
			"addr": 4,
			"dataType": "Boolean",
			"wordOrder": "abcd"
		},
		"ioManager": {
			"tag": "101_LSL_111",
			"qualification": { "kind": "DI", "configuration": { "direct": true } }
		}
	},
	{
		"name": "PU131 motor current",
		"address": {
			"protocol": "modbus_tcp",
			"registerType": "input",
			"addr": 20,
			"dataType": "UInt16",
			"wordOrder": "abcd"
		}
	},
	{
		"name": "Spare",
		"address": {
			"protocol": "modbus_tcp",
			"registerType": "input",
			"addr": 21,
			"dataType": "UInt16",
			"wordOrder": "abcd"
		}
	}
]
```

In the final report, taken from the apply's `rows[]`, the first line names
its point and channel actions; `PU131 motor current` and `Spare` are named as
**raw points** (`channelAction: "none"`), not left out.
