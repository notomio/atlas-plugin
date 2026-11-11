---
name: import-register-map
description: Use when an Atlas employee wants to turn a register table — a Modbus register map, a remote I/O card datasheet, an OPC-UA node list — into the raw points and confirmed IO Manager channels of a data source already linked to an Interact PLC. Triggers on "import this register map", "declare the points of this card", "seed the Modbus points from the datasheet", "importe la table de registres", "déclare les points de cette carte", or "crée les canaux depuis cette table". Never creates or links a data source.
---

# Import a register table into a linked data source

Walk one register table from the document to confirmed points and channels:
document → `rows[]` → preview → confirmation → apply → report, one batch at a
time.

First follow the [connector and recovery procedure](../configure-plc/references/session-and-recovery.md), retaining the chosen connector for every call. Render every read, effect and refusal as [restitution](../configure-plc/references/restitution.md) prescribes. Compose each line with the [register-map reference](references/register-map.md). Read a label's letters with [ISA-5.1 letters and equipment codes](../interact-guide/references/isa-5-1-tags.md) — a suggestion to confirm against the site convention, never a fact. What a kind means is in [signal meaning and selection](../interact-guide/references/signals.md).

## When to use

The user supplies a register table and wants its lines declared on an Interact data source. The source must already be `modbus_tcp` or `opcua` and already linked to the PLC. Creating or linking it belongs to [link-plc-data-source](../link-plc-data-source/SKILL.md). Correcting one existing point name, address or channel tag belongs to the targeted update tools in [bindings](../configure-equipment/references/bindings.md) § Adjust a point or channel — never to a sync.

## Stop before composing

Settle the entry before reading a single line of the table. Until it holds, call no sync tool with rows, and create nothing.

1. **Resolve the site and the PLC.** Use the provided ids, or `atlas_isa95_list_sites` then `atlas_interact_plc_list`. Ask which PLC when several match. An empty `plcs` list means the PLC must first be created in Atlas: stop.
2. **Resolve one `dataSourceId` among that PLC's `dataSourceIds`.** An id the user gives, or one confirmed as [link-plc-data-source](../link-plc-data-source/SKILL.md) step 3 resolves a source, must appear in that list. Never pick one by rank or name alone.
3. **Confirm its protocol** with `atlas_interact_preview_points_sync`, passing the `dataSourceId` and `rows: []`. It writes nothing and returns the source's `protocol`, `modbus_tcp` or `opcua`.

If there is no linked source, the source is linked to another PLC, or the preview answers `NOT_FOUND` or `UNSUPPORTED_PROTOCOL`: say which of these holds and send the user to [link-plc-data-source](../link-plc-data-source/SKILL.md). Do not create a source, a link, a point or a channel from this skill, and do not continue the import.

## Recipe

1. **Read what the source already holds.** Call `atlas_interact_list_channels` for the `dataSourceId` and collect every page. It lists the qualified points — name, address, channel tag. Raw points without a channel are absent from it; the preview reveals them later by name, as `update` or `unchanged`.
2. **Read the document on the client.** The document never travels to the server: only structured `rows[]` do. Identify its columns — register type or node id, address, data type, label, unit, bounds. Name what does not read — a missing column, an ambiguous line, an unreadable scan, a bit inside a word register — and stop on that point instead of completing it. A gap is a result.
3. **Ask what the table does not fix, before composing any row.** Word order, addressing base, a word register's data type, an OPC-UA node's direction, a digital line's polarity, an analog line's scaling or raw mode: ask for each one the table leaves open, as listed in the [register-map reference](references/register-map.md). Assume none of them, and do not convert a `40001`-style address without the base the user confirmed.
4. **Propose each line, then wait for its confirmation.** Follow [restitution](../configure-plc/references/restitution.md) § Proposing a register line: **at most one kind**, said in plant words from the register or node capability and the document's hints.
   - Check the proposed tag against the IEC rule in the [register-map reference](references/register-map.md). If `IO_<tag>` is not an IEC 61131-3 identifier, propose the corrected tag, show the original beside it, and wait. Never rename a tag, or any other value, silently.
   - Check the name and unit against the limits in the [register-map reference](references/register-map.md): a single malformed value rejects the whole call. A label that does not fit is proposed shortened and confirmed, never truncated.
   - A confirmed line carries `ioManager`. A line the user declines, leaves unanswered or keeps unqualified carries none. It stays a **raw point** if its point has no channel yet; the sync never removes an existing channel.
   - A line whose address or node matches an existing point under another name is a rename, and the sync cannot rename: sent as is, it creates a second point on the same OPC-UA node, or is refused `ADDRESS_OVERLAP` on Modbus. Say so and ask — keep the stored name, or correct it first with the targeted tools.
5. **Compose and split into batches** of at most 100 rows, after checking that no name repeats anywhere in the table. Say that each batch is a separate call and that nothing is transactional across batches: a later refusal never undoes an earlier batch.
6. **Preview one batch** with `atlas_interact_preview_points_sync`, only once the previous batch is applied. Render `totals` — `create`, `update`, `unchanged`, `error` — then every `update` row with what it changes (address, tag, qualification, a unit erased or an entry reactivated by an omitted field) — compared with the source read, or, for a raw point that read does not list, flagged as overwriting stored values this session has not seen, then every row with a `pointErrorCode` or `channelErrorCode`, by its name and code. Say what this preview cannot predict: a tag already carried by a stored channel of the site, including one of this source, and a point or channel already wired. Correct refused rows with the user, then preview the batch again.
7. **Apply a batch only after the user confirms that batch**, with `atlas_interact_apply_points_sync` and exactly the previewed rows. Confirmation of one batch never covers the next. After a timeout or lost connection the outcome is unknown: follow the [recovery procedure](../configure-plc/references/session-and-recovery.md) and preview that batch again before any new apply. If a call is rejected as a whole for invalid input, nothing of it was written: find the value, ask, then preview again.
8. **Report from the apply result, row by row.** Read `rows[]` of the apply, never the preview: for each line, the point action, the channel action and any refusal code, as the [register-map reference](references/register-map.md) reads them. Name every refused line, every channel `skipped` because its point failed, and every **raw point** — a line sent without `ioManager` whose point succeeded and had no channel. A line sent without `ioManager` on a point that already had a channel keeps that channel: say so. Then give `applied` and `noop`.

## Guardrails

- **Nothing unseen is written.** Every row sent was shown in its rendered form and confirmed, batch by batch. No automatic application, at any step.
- **No default is invented.** Word order, addressing base, data type, direction, polarity and scaling come from the document or the user.
- **One kind at most, never from letters alone.** The backend matrix refuses the impossible (`IO_KIND_MISMATCH`); the plausible-but-wrong — a measurement on a writable holding register qualified as a command — is caught only by the confirmation.
- **Additive and idempotent.** The sync never deletes a point or channel absent from the payload, and replaying a batch in place reports `noop: true`. It upserts on the point `name`, so it cannot rename a point.
- **Not a correction path.** For a point name, an address or a tag on an existing entry, use the targeted update tools of [bindings](../configure-equipment/references/bindings.md) § Adjust a point or channel.
- **Declaring is not collecting.** Points and channels reach the machine only once the PLC project is generated and deployed. Do not say data is flowing, and do not generate, publish or deploy from this skill.
